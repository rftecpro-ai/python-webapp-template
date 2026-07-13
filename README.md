# python-webapp-template

A minimal Flask "Hello World" web app, containerized with Docker and built/pushed
to Docker Hub via a GitHub Actions CI pipeline. Deployed to Kubernetes via
Kustomize and ArgoCD GitOps. This is the Kustomize-based sibling of
[python-webapp-helm-template](https://github.com/rftecpro-ai/python-webapp-helm-template),
which uses a Helm chart instead.

## Repository structure

```
.
├── app.py                        # Flask app (single "/" route, returns JSON)
├── conftest.py                   # empty; makes pytest resolve `app` as an importable module
├── tests/
│   └── test_app.py               # pytest suite for app.py
├── requirements.txt               # runtime dependencies (flask, gunicorn)
├── requirements-dev.txt           # requirements.txt + pytest, for local dev/CI
├── Dockerfile                     # multi-arch image build, served via gunicorn
├── VERSION                        # human-edited release version (see CI/CD below)
├── .github/workflows/ci.yml       # test → build-and-push → update-manifest pipeline
├── base/                          # shared Kustomize base (Deployment + Service)
│   ├── kustomization.yaml          # lists base resources: deployment.yaml, service.yaml
│   ├── deployment.yaml             # Deployment (2 replicas, container port 8000)
│   └── service.yaml                # ClusterIP Service, port 80 → container port 8000
├── overlays/
│   └── dev/
│       └── kustomization.yaml     # dev overlay: sets namespace + deployed image tag
├── applications/
│   └── argocd-app.yaml            # ArgoCD Application CRD, registers overlays/dev with ArgoCD
├── .dockerignore                  # excludes tests/, base/, overlays/, applications/, docs, etc. from build context
├── .gitignore                     # excludes __pycache__/, .venv/, .pytest_cache/, etc.
└── README.md
```

Not shown above: `__pycache__/` and `.pytest_cache/` are ephemeral, gitignored
directories that Python/pytest regenerate locally — they're not part of the
repo. `.git/` is version-control internals.

See [CI/CD](#cicd) and [Deployment (ArgoCD / Kubernetes)](#deployment-argocd--kubernetes)
below for how these pieces fit together.

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
   [`overlays/dev/kustomization.yaml`](overlays/dev/kustomization.yaml) to
   `:<commit-sha>` and commits that change back to `main` (with `[skip ci]`,
   so it doesn't re-trigger the pipeline). This is what drives the ArgoCD
   deployment below — every push to `main` results in that exact commit's
   image running in the cluster.

To cut a new Docker Hub release version, bump the version in the
[`VERSION`](VERSION) file and merge to `main` — that value becomes the
`:<VERSION>` image tag. Without a bump, every merge to `main` re-pushes the
same version tag (pointing at the latest commit), so treat updating `VERSION`
as the release step. This is independent of the `:<commit-sha>` tag that
`update-manifest` deploys, which always tracks the latest commit on `main`.

`VERSION` is the **only** file you need to edit for a release. It's tempting
to think the image tags in [`base/deployment.yaml`](base/deployment.yaml) and
[`overlays/dev/kustomization.yaml`](overlays/dev/kustomization.yaml) also need
a manual bump, but they don't:

- `base/deployment.yaml`'s image tag is never deployed as-is — Kustomize's
  `images` transformer in the overlay always overwrites it. It only needs to
  keep the correct image *name*, not a correct tag.
- `overlays/dev/kustomization.yaml`'s `newTag` is overwritten automatically by
  `update-manifest` on every merge to `main`, using the commit SHA — not the
  `VERSION` value. Any manual edit there gets clobbered on the next merge
  anyway.

### Required repository secrets

The push step needs these secrets set in the GitHub repo
(**Settings → Secrets and variables → Actions**):

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
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

Kubernetes manifests are laid out as a Kustomize base + per-environment
overlays, so adding another environment later (e.g. `prod`) only means adding
another overlay, not duplicating the whole manifest set:

```
.
├── base/                    # shared Deployment + Service, environment-agnostic
├── overlays/
│   └── dev/                 # patches base for the dev environment (namespace, image tag)
└── applications/
    └── argocd-app.yaml      # ArgoCD Application CRD, points at overlays/dev
```

| Path | Purpose |
|---|---|
| [`base/kustomization.yaml`](base/kustomization.yaml) | Lists the base resources (`deployment.yaml`, `service.yaml`) |
| [`base/deployment.yaml`](base/deployment.yaml) | Deployment (2 replicas, container port 8000) |
| [`base/service.yaml`](base/service.yaml) | ClusterIP Service, port 80 → container port 8000 |
| [`overlays/dev/kustomization.yaml`](overlays/dev/kustomization.yaml) | Sets the `dev` namespace and the deployed image tag |
| [`applications/argocd-app.yaml`](applications/argocd-app.yaml) | ArgoCD `Application` CRD, watches `overlays/dev` |

ArgoCD auto-detects the Kustomize overlay from the `kustomization.yaml` at the
watched path — no extra ArgoCD-side config needed. Sync policy is automated
with `selfHeal: true` and `prune: true` — this repo is the source of truth.
Manual changes made directly against the cluster will be reverted, and
resources removed from the overlay will be deleted from the cluster.

**One-time bootstrap** (run once against your cluster to register the app —
not something I can run for you, since it targets your local lab):

```bash
kubectl apply -f applications/argocd-app.yaml
```

After that, ArgoCD manages everything: CI's `update-manifest` job updates
`overlays/dev/kustomization.yaml`'s image tag on every push to `main`, ArgoCD
detects the drift, and syncs it to the cluster automatically.

**Infrastructure changes** (replicas, resource limits): edit
[`base/deployment.yaml`](base/deployment.yaml) directly — it's shared across
all overlays — and merge to `main`. **Per-environment changes** (namespace,
replica overrides, etc. for `dev` specifically): edit
[`overlays/dev/kustomization.yaml`](overlays/dev/kustomization.yaml) instead.
Don't `kubectl apply` by hand either way, ArgoCD will just revert it.
