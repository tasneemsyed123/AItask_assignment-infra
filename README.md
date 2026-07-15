# k8s/README.md

Deploys the AI Task Platform (mongo, redis, backend, worker, frontend) to a
local minikube cluster. This file is referenced by comments in several of the
manifests (`imagePullPolicy: Never`, build commands, common pitfalls) - keep
it in sync if those change.

## Prerequisites

- Docker Desktop (WSL2 backend on Windows)
- minikube, kubectl
- cloudflared (optional - only needed for a public URL, see "Exposing it
  publicly" below)

### Windows/WSL2 memory - read this before starting

Docker Desktop's Resources > Advanced memory slider does **not** reliably
control the WSL2 VM's memory ceiling. Without an explicit `.wslconfig`, WSL2
defaults to 50% of the host's total RAM, no matter what the Docker Desktop UI
shows. On an 8GB machine that's ~3.75GB shared across minikube +
ingress-nginx + metrics-server + all 5 app pods, which is not enough - it
manifests as mongo taking 60-90s to recover on every boot and eventually
crashing with a `missing featureCompatibilityVersion document` fatal
assertion (see "Common pitfalls" below).

Create `%UserProfile%\.wslconfig` (e.g. `C:\Users\<you>\.wslconfig`):

```ini
[wsl2]
memory=5GB
processors=4
swap=4GB
```

Adjust `memory` to your machine's actual RAM (leave at least 2.5-3GB for
Windows itself). Then apply it:

1. Quit Docker Desktop completely (tray icon -> Quit, not just closing the window)
2. `wsl --shutdown`
3. Reopen Docker Desktop and wait for it to finish starting
4. `minikube start` (a `wsl --shutdown` stops minikube's container too)

Verify it took effect: `docker info` should show the new memory figure.

## First-time setup

```
minikube start
minikube addons enable metrics-server   # required for the worker HPA
```

(`ingress`, `storage-provisioner`, and `default-storageclass` are usually
already enabled by default in recent minikube versions - `minikube addons
list` to check.)

### Secrets

`02-secret.yaml` is gitignored and does not exist until you create it:

```
cp 02-secret.example.yaml 02-secret.yaml
```

Fill in real values (generate with
`node -e "console.log(require('crypto').randomBytes(24).toString('hex'))"`).
Reusing the same values already generated for `project/.env` /
`backend/.env` is fine - this is a separate Mongo/Redis instance with its own
empty volume.

### Build and load the images

minikube can't pull these from a registry (`imagePullPolicy: Never`), so
build them locally and load them into the cluster's image store:

```
# from project/backend
docker build -t ai-task-platform-backend:k8s .

# from project/worker
docker build -t ai-task-platform-worker:k8s .

# from project/frontend - NOTE the build-arg: a RELATIVE path, not the
# absolute localhost:4000 URL docker-compose uses. The browser reaches this
# app through whatever host the Ingress/tunnel is running on that session
# (minikube IP, port-forward, or a fresh *.trycloudflare.com URL each time),
# and a relative path resolves correctly regardless of which one it is.
docker build --build-arg NEXT_PUBLIC_API_BASE_URL=/api/v1 -t ai-task-platform-frontend:k8s .

minikube image load ai-task-platform-backend:k8s
minikube image load ai-task-platform-worker:k8s
minikube image load ai-task-platform-frontend:k8s
```

Re-run the relevant `docker build` + `minikube image load` pair whenever you
change backend/worker/frontend source - there's no automatic rebuild.

### Apply the manifests

The numeric filename prefixes are the safe apply order (namespace before
anything that references it, secrets/configmaps before the pods that consume
them, mongo's init-script configmap before the mongo deployment that mounts
it):

```
kubectl apply -f .
```

Or apply individually in order (00 through 09) if you want to watch each
step. Check status with:

```
kubectl get pod -n ai-task-platform
kubectl get all -n ai-task-platform
```

Everything should reach `1/1` or `2/2` Running (worker starts at 2 replicas).

## Common pitfalls

**Mongo crash-looping with a cryptic `missing featureCompatibilityVersion
document` fatal assertion**, not an obvious OOMKilled status. Root cause is
almost always insufficient memory on the node (see the WSL2 section above) -
mongod gets killed mid-init or mid-recovery repeatedly, and each unclean
shutdown makes the next recovery slower, until one kill lands after
WiredTiger opens but before the FCV document is written, which is
unrecoverable short of `--repair` or wiping the volume. If you hit this on a
fresh volume: fix the underlying memory constraint first, then

```
kubectl delete deployment mongo -n ai-task-platform
kubectl delete pvc mongo-data -n ai-task-platform
kubectl apply -f 04-mongo.yaml
```

(This deletes all data in the dev Mongo instance - fine for local dev, not
something to do against real data.)

**Do not add `--auth` to mongo's container `args`.** It looks like the
obviously-correct thing to do given `MONGO_INITDB_ROOT_USERNAME/PASSWORD` are
set, but this mongo image already enables authorization automatically once
those are set (confirmed from a live pod's own startup logs - `"security":
{"authorization":"enabled"}` appears even without an explicit `--auth` flag).
Passing `--auth` explicitly interferes with the entrypoint script's
first-boot bootstrap sequence (the temporary unauthenticated instance it uses
to create the root user and write internal setup documents) and reliably
reproduces the featureCompatibilityVersion crash above, even on a completely
empty volume.

**Probe timeouts on a resource-constrained node.** The default 3-5s
`timeoutSeconds` on the HTTP and exec probes assumes reasonably free CPU.
Under load (a rolling deploy, `kubectl top` etc. all fighting for the same
few cores) they can intermittently time out and log `Unhealthy` events
without necessarily restarting the pod (that needs `failureThreshold`
consecutive failures). Current manifests already use widened timeouts
(15s for mongo/redis exec probes, 10s for backend/frontend HTTP probes) to
tolerate this; if you still see pods actually restarting (not just probe
warnings) rather than just recovering, widen further or check `kubectl top
node` for a real capacity problem.

**Mongo's one restart on first boot is expected.** After `kubectl apply -f
04-mongo.yaml` on a fresh volume, expect exactly one restart with `exitCode:
0, reason: "Completed"` shortly after the pod starts - that's the official
mongo image's own bootstrap process (temporary instance creates the root
user + runs `03-mongo-init-configmap.yaml`'s app-user script, then exits
cleanly; Kubernetes restarts the container into the real long-running
mongod). Only worry if the restart count keeps climbing after that.

## Accessing the app

The Ingress has no `host` rule (matches any Host header), so reach it via a
local port-forward to the ingress controller rather than a fixed hostname:

```
kubectl port-forward -n ingress-nginx svc/ingress-nginx-controller 8080:80
```

Then `http://localhost:8080` serves the frontend, and `http://localhost:8080/api/*`
routes to the backend. `minikube tunnel` is not needed for this - the
port-forward goes through the Kubernetes API server directly, independent of
minikube's driver/networking setup.

### Exposing it publicly (optional)

```
cloudflared tunnel --url http://localhost:8080
```

prints a random `https://<words>.trycloudflare.com` URL. It's an anonymous
"quick tunnel" - no uptime guarantee, and the URL changes every time
cloudflared restarts. Because that shared domain is also used by real
phishing campaigns (it's free and requires no account), Chrome's Safe
Browsing sometimes flags a given random subdomain even when it's pointing at
your own harmless local app - that's a false positive on the domain's shared
reputation, not a sign anything is actually wrong. If it happens, either
retry for a new random subdomain or just use the local `http://localhost:8080`
URL, which has no such issue. For anything beyond a quick demo, set up a
proper named Cloudflare Tunnel against your own domain instead.

## Useful commands

```
kubectl get pod -n ai-task-platform                                   # health snapshot
kubectl get events -n ai-task-platform --sort-by='.lastTimestamp'      # recent problems
kubectl logs -n ai-task-platform -l app=<mongo|redis|backend|worker|frontend>
kubectl describe pod -n ai-task-platform <pod-name>                    # why a pod won't start
kubectl top pod -n ai-task-platform                                    # CPU/memory per pod
kubectl top node                                                       # is the node actually out of capacity
```
