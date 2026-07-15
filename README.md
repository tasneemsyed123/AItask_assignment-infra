# AI Task Platform — Infra

GitOps deployment config for the AI Task Platform (mongo, redis, backend,
worker, frontend). ArgoCD watches this repo and keeps a live Kubernetes
cluster in sync with whatever is committed here — this is the **infra**
repo, kept separate from the **application code** repo on purpose (see
"Why two repos" below).

## Live deployment

- **App**: http://34.93.242.6/
- **ArgoCD UI**: https://34.93.242.6:32121/ (self-signed cert — browser will
  warn, click through Advanced → Proceed)
  - Username: `admin`
  - Password: not committed here. Retrieve it with:
    ```
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
    ```
    (This is the *initial* password — change it after first login via the
    ArgoCD UI or CLI, since anyone who reads this repo's history could see
    the retrieval command, though not the password itself.)
- **Code repo**: https://github.com/tasneemsyed123/AItask_assignment
- **Infra repo**: https://github.com/tasneemsyed123/AItask_assignment-infra (this one)

## Architecture

```
GitHub (this repo)
      |  ArgoCD polls / webhook
      v
Google Cloud VM (e2-medium, 4GB RAM)
      |
      +-- k3s (lightweight Kubernetes)
            |
            +-- ingress-nginx  --- routes :80/:443 traffic
            |                       |-- /api/*  -> backend Service
            |                       `-- /*      -> frontend Service
            |
            +-- ArgoCD  --- watches this repo, applies changes automatically
            |
            `-- ai-task-platform namespace
                  |-- mongo (PVC-backed, auth enabled by the image itself)
                  |-- redis (PVC-backed, password-protected)
                  |-- backend  (image: docker.io/tasneemsyed/ai-task-platform-backend)
                  |-- worker x2, HPA-scaled 2-5 (image: .../ai-task-platform-worker)
                  `-- frontend (image: .../ai-task-platform-frontend)
```

Images are pulled from **Docker Hub** (`docker.io/tasneemsyed/ai-task-platform-*`),
not built on the cluster. Build locally, push, update the image tag in this
repo, commit — ArgoCD does the rest.

## Why two repos

The assessment asks for a code URL and a live URL as separate deliverables.
Splitting app source from deployment config is also the standard GitOps
pattern for a real reason: a routine code commit (which triggers a CI build)
and a deployment change (which triggers ArgoCD to touch the live cluster)
are different kinds of events, and keeping them in separate repos means one
never accidentally triggers the other. For a project this size that
separation is somewhat more ceremony than strictly necessary, but it's what
was asked for.

## One-time setup — provisioning the cluster from scratch

If this VM is ever rebuilt, here's the exact sequence used to get from a
blank Ubuntu 22.04 VM to the running state above.

### 1. VM

Google Cloud, Compute Engine → Create Instance:
- Machine type: **e2-medium** (2 vCPU / 4GB RAM) — a free-tier `e2-micro`
  (1GB) is not enough; expect the same OOM crash-loop behavior documented
  in the "Common pitfalls" section below, just worse.
- Boot disk: Ubuntu 22.04 LTS, 30GB (default 10GB is too tight)
- Firewall: allow HTTP and HTTPS traffic (checkboxes at creation time)

**Firewall rule for NodePort services** (needed for ArgoCD's UI, exposed via
NodePort since it doesn't share ingress-nginx's ports 80/443):
VPC Network → Firewall → Create Firewall Rule — target tag `allow-nodeports`,
source `0.0.0.0/0`, TCP `30000-32767`. Then add that same tag to the VM
instance (Edit → Network tags). This opens the whole NodePort range rather
than one port at a time — a reasonable tradeoff for a disposable
demo/assessment VM, not something to do on real production infra without
narrowing it down.

**Passwordless sudo**, needed for any automated (non-interactive SSH)
command to run `sudo` without hanging on a password prompt:
```
echo "<your-username> ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/91-nopasswd
sudo chmod 440 /etc/sudoers.d/91-nopasswd
```

### 2. k3s

```
curl -sfL https://get.k3s.io | sh -s - --disable traefik
```

`--disable traefik` skips k3s's bundled ingress controller in favor of
ingress-nginx (installed next), so behavior matches what's already been
validated in local (minikube) testing. k3s's other bundled components —
`servicelb` (a lightweight load balancer that binds host ports directly,
which is what lets a plain `LoadBalancer`-type Service get an external IP
on a single bare VM with no cloud provider integration), `local-path-provisioner`
(default StorageClass), `coredns`, and `metrics-server` — are all left enabled
and used as-is.

### 3. ingress-nginx

```
sudo k3s kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.3/deploy/static/provider/cloud/deploy.yaml
```

The "cloud" provider variant creates a `LoadBalancer`-type Service, which
works here specifically because k3s's `servicelb` satisfies it (see above) —
on vanilla upstream Kubernetes with no cloud integration this would sit in
`<pending>` forever, so if this is ever migrated off k3s, switch to the
"baremetal" provider manifest (NodePort-based) instead.

### 4. ArgoCD

```
sudo k3s kubectl create namespace argocd
sudo k3s kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.13.2/manifests/install.yaml
```

Expose the UI. Routing it through ingress-nginx would need either a real
domain (for host-based routing) or ArgoCD's subpath support (fragile, not
worth it for a bare-IP setup), and it can't share ports 80/443 as a second
`LoadBalancer` Service (those are already claimed by ingress-nginx's
`servicelb` daemonset). Simplest: NodePort on its own port.

```
sudo k3s kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
sudo k3s kubectl get svc argocd-server -n argocd   # note the assigned port, e.g. 32121
```

Get the initial admin password (see "Live deployment" above for the exact
command).

### 5. Application namespace + secrets

The namespace and `app-secret` (real DB/JWT credentials) are **not** in this
repo — deliberately, same reasoning as the local minikube setup's gitignored
`02-secret.yaml`. Apply them by hand once:

```
kubectl apply -f 00-namespace.yaml -f <your-real-secret-file>.yaml
```

### 6. The ArgoCD Application

This is what actually tells ArgoCD to watch this repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ai-task-platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/tasneemsyed123/AItask_assignment-infra.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: ai-task-platform
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

`automated` + `selfHeal` means ArgoCD applies changes from git automatically
(no manual "sync" click needed) and reverts any manual `kubectl edit` drift
back to match git — the cluster's state always follows what's committed
here, which is the entire point of GitOps.

## Building and pushing images

Images aren't built on the cluster — build locally, push to Docker Hub,
bump the tag in this repo.

```
# from project/backend
docker build -t tasneemsyed/ai-task-platform-backend:latest .
docker push tasneemsyed/ai-task-platform-backend:latest

# from project/worker
docker build -t tasneemsyed/ai-task-platform-worker:latest .
docker push tasneemsyed/ai-task-platform-worker:latest

# from project/frontend — see the Windows/Git Bash warning below before running this one
docker build --build-arg NEXT_PUBLIC_API_BASE_URL=/api/v1 -t tasneemsyed/ai-task-platform-frontend:latest .
docker push tasneemsyed/ai-task-platform-frontend:latest
```

Then update the `image:` line for whichever service changed in
`06-backend.yaml` / `07-worker.yaml` / `08-frontend.yaml`, commit, push.
ArgoCD picks it up on its next poll (~3 min) or immediately via:

```
kubectl -n argocd annotate application ai-task-platform argocd.argoproj.io/refresh=hard --overwrite
```

**Prefer a versioned tag over `latest`** when you want to guarantee a
redeploy. `imagePullPolicy: IfNotPresent` means a node that already has an
image cached under a given tag won't re-check the registry just because the
tag's content changed remotely — pushing a new `:latest` and re-syncing
won't actually pull the new bits if the old `:latest` is still sitting in
the node's local image store. A fresh tag (`:v2`, a commit SHA, etc.)
sidesteps this entirely.

### Critical: never build the frontend image via Git Bash on Windows

Git Bash (MSYS2) silently rewrites any command-line argument that looks
like a POSIX path (starts with `/`) into a Windows path before the command
ever runs. `--build-arg NEXT_PUBLIC_API_BASE_URL=/api/v1` becomes
`NEXT_PUBLIC_API_BASE_URL=C:/Program Files/Git/api/v1` — Docker receives the
mangled value, bakes it straight into the Next.js client bundle (it's a
build-time constant, not a runtime env var — see the frontend Dockerfile's
comments), and every API call the browser makes then targets that garbage
URL. It fails **client-side, silently** — no request ever reaches the
backend, so backend logs show nothing, curl-based testing of the API
directly looks completely fine, and only an actual browser hitting the
actual frontend reveals the problem. This exact bug shipped once already
and took real debugging time to trace.

**Build the frontend from PowerShell or cmd.exe, never Git Bash.** If you
must use Git Bash, prefix the path with a second slash (`//api/v1`) or set
`MSYS_NO_PATHCONV=1` for that command — but PowerShell is the simpler,
foolproof option. Verify after building, before pushing:

```powershell
docker run --rm tasneemsyed/ai-task-platform-frontend:<tag> sh -c "grep -roE '.{20}baseURL:\"[^\"]*\".{5}' /app/.next/static/chunks/app/register/page-*.js"
```

should print `baseURL:"/api/v1"` — anything containing `Program Files` or
`C:` means it's broken.

## Common pitfalls

**Mongo crash-looping with a cryptic `missing featureCompatibilityVersion
document` fatal assertion**, not an obvious OOMKilled status. Root cause is
almost always insufficient memory on the node — mongod gets killed mid-init
or mid-recovery repeatedly, and each unclean shutdown makes the next
recovery slower, until one kill lands after WiredTiger opens but before the
FCV document is written, which is unrecoverable short of `--repair` or
wiping the volume. This was hit repeatedly during local minikube testing on
an 8GB Windows laptop (see the equivalent section in `project/k8s/README.md`
for the full WSL2/.wslconfig story) — it's much less likely on this VM's
dedicated 4GB, but if it happens:

```
kubectl delete deployment mongo -n ai-task-platform
kubectl delete pvc mongo-data -n ai-task-platform
kubectl apply -f 04-mongo.yaml
```

(This deletes all data in the Mongo instance.)

**Do not add `--auth` to mongo's container `args`.** It looks like the
obviously-correct thing to do given `MONGO_INITDB_ROOT_USERNAME/PASSWORD`
are set, but this mongo image already enables authorization automatically
once those are set (confirmed from a live pod's own startup logs —
`"security":{"authorization":"enabled"}` appears even without an explicit
`--auth` flag). Passing `--auth` explicitly interferes with the entrypoint
script's first-boot bootstrap sequence and reliably reproduces the FCV
crash above, even on a completely empty volume.

**Mongo's one restart on first boot is expected.** After `kubectl apply -f
04-mongo.yaml` on a fresh volume, expect exactly one restart with `exitCode:
0, reason: "Completed"` shortly after the pod starts — that's the official
mongo image's own bootstrap process (a temporary instance creates the root
user + runs the app-user creation script, then exits cleanly; Kubernetes
restarts the container into the real long-running mongod). Only worry if
the restart count keeps climbing after that.

**ArgoCD reachable but "sync" shows `OutOfSync`/`Missing` right after
creating the Application.** Normal for the first few seconds — it hasn't
done its first comparison yet. If it stays that way, check
`kubectl -n argocd logs deploy/argocd-repo-server` for a repo access error
(wrong URL, private repo needing credentials this Application doesn't have,
etc.)

**A service account/VM without `cloud-platform` scope can't manage GCP
resources from inside the VM** (e.g. `gcloud compute firewall-rules create`
fails with "insufficient authentication scopes") even though `gcloud` itself
is pre-installed and authenticated as the VM's own service account. Firewall
rules, IAM changes, etc. need to be done from the Cloud Console (or a
gcloud session with a user identity, not the VM's default service account)
unless the VM was explicitly created with broader OAuth scopes.

## Local development

For local development against minikube instead of this live cluster (faster
iteration, no need to push images anywhere), see the parallel setup docs in
the application code repo at `project/k8s/README.md` — same manifests,
different (local, `imagePullPolicy: Never`) image-loading workflow instead
of Docker Hub, and a `cloudflared` quick-tunnel option instead of a real
public IP.

## Useful commands

```
kubectl get application ai-task-platform -n argocd            # sync/health status
kubectl get pod -n ai-task-platform                           # app health snapshot
kubectl get events -n ai-task-platform --sort-by='.lastTimestamp'
kubectl logs -n ai-task-platform -l app=<mongo|redis|backend|worker|frontend>
kubectl describe pod -n ai-task-platform <pod-name>            # why a pod won't start
kubectl top pod -n ai-task-platform                            # CPU/memory per pod
kubectl -n argocd annotate application ai-task-platform argocd.argoproj.io/refresh=hard --overwrite
                                                                # force an immediate re-sync
```

## Security follow-ups worth doing

- Rotate the ArgoCD admin password from its initial auto-generated value
- Put a real TLS cert on both the app and ArgoCD (currently self-signed —
  browsers warn on the ArgoCD UI, and the app itself is plain HTTP)
- Narrow the `allow-nodeports` firewall rule down to just ArgoCD's actual
  port once it's known and stable, instead of the whole 30000-32767 range
- Revoke/rotate the Docker Hub access token used to push images if it was
  ever shared outside this environment
