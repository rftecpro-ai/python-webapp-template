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

To cut a new release, bump the version in the [`VERSION`](VERSION) file and
merge to `main` — that value becomes the image tag. Without a bump, every
merge to `main` re-pushes the same version tag (pointing at the latest
commit), so treat updating `VERSION` as the release step.

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