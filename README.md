# python-webapp-template

A minimal Flask "Hello World" web app, containerized with Docker and built/pushed
to Docker Hub via a GitHub Actions CI pipeline.

## Local development

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements-dev.txt

python app.py          # runs on http://localhost:8000
pytest                 # run tests
```

## Docker

```bash
docker build -t ffribeiro/python-webapp-template:local .
docker run -p 8000:8000 ffribeiro/python-webapp-template:local
```

## CI/CD

`.github/workflows/ci.yml` runs on every push/PR to `main`:

1. **test** — installs dependencies and runs `pytest`.
2. **build-and-push** — on push to `main` only (after tests pass), builds a
   multi-arch (`linux/amd64`, `linux/arm64`) image and pushes it to Docker Hub
   as `ffribeiro/python-webapp-template:<VERSION>` and `:<commit-sha>`.
3. **update-manifest** — after the image is pushed, sets the image tag in
   [`dev/deployment.yaml`](dev/deployment.yaml) to `:<commit-sha>` and commits
   that change back to `main` (with `[skip ci]`, so it doesn't re-trigger the
   pipeline). This is what drives the ArgoCD deployment below — every push to
   `main` results in that exact commit's image running in the cluster.

To cut a new Docker Hub release version, bump the version in the
[`VERSION`](VERSION) file and merge to `main` — that value becomes the
`:<VERSION>` image tag. Without a bump, every merge to `main` re-pushes the
same version tag (pointing at the latest commit), so treat updating `VERSION`
as the release step. This is independent of the `:<commit-sha>` tag that
`update-manifest` deploys, which always tracks the latest commit on `main`.

### Required repository secrets

The push step needs these secrets set in the GitHub repo
(**Settings → Secrets and variables → Actions**):

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username (`ffribeiro`) |
| `DOCKERHUB_TOKEN` | A Docker Hub [access token](https://hub.docker.com/settings/security) (not your password) |

I can't create these secrets for you (I don't have your Docker Hub credentials) —
add them manually before merging to `main`, otherwise the `build-and-push` job
will fail at the login step.

### Required repository permissions

`update-manifest` pushes a commit back to this repo using the default
`GITHUB_TOKEN`. If that push fails with a permissions error, enable
**Settings → Actions → General → Workflow permissions → Read and write
permissions** for this repo.

## Deployment (ArgoCD / Kubernetes)

[`dev/`](dev) holds the Kubernetes manifests ArgoCD watches for the `dev`
environment:

| File | Purpose |
|---|---|
| [`application.yaml`](dev/application.yaml) | ArgoCD `Application` CRD registering this repo/path with ArgoCD |
| [`deployment.yaml`](dev/deployment.yaml) | Deployment (2 replicas, container port 8000) |
| [`service.yaml`](dev/service.yaml) | ClusterIP Service, port 80 → container port 8000 |

Sync policy is automated with `selfHeal: true` and `prune: true` — this repo's
`dev/` directory is the source of truth. Manual changes made directly against
the cluster will be reverted, and resources removed from `dev/` will be
deleted from the cluster.

**One-time bootstrap** (run once against your cluster to register the app —
not something I can run for you, since it targets your local lab):

```bash
kubectl apply -f dev/application.yaml
```

After that, ArgoCD manages everything: CI's `update-manifest` job updates
`dev/deployment.yaml`'s image tag on every push to `main`, ArgoCD detects the
drift, and syncs it to the cluster automatically.

**Infrastructure changes** (replicas, ports, resource limits): edit the
manifests under `dev/` directly and merge to `main` — do not `kubectl apply`
by hand, ArgoCD will just revert it.