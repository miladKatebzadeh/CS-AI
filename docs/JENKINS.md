# CS.AI — Jenkins CI/CD

This repository includes a root `Jenkinsfile`. It builds and tests **CS.AI**, a personal fork of Open WebUI, using the Dockerfile checked into the repository. Docker images are built **locally on the Jenkins Docker host**; the pipeline does not publish packages or images.

## Requirements

- Jenkins with the **Pipeline** and **Git** plugins
- A **Linux Jenkins agent** with `git`, `docker` CLI and a functioning **Docker Engine**
- Jenkins agent permission to access the Docker daemon (e.g. authorized Docker socket access; note that Docker access is effectively host-administrator access)
- Internet access for Docker image pulls and project dependency installs
- Recommended for a first build: **16 GB RAM** and significant free disk space (especially if `SLIM_IMAGE=false`). The frontend build allows a Node.js heap of up to 12 GB.

## Create a Jenkins job

1. In Jenkins, select **New Item** → **Pipeline**; name it `CS-AI`.
2. Under **Pipeline**, choose **Pipeline script from SCM**.
3. Set **SCM = Git**.
4. Enter **Repository URL**: `https://github.com/miladKatebzadeh/CS-AI.git`.
5. Set **Branches to build**: `*/main`.
6. Set **Script Path**: `Jenkinsfile`.
7. Save, then select **Build with Parameters**. Public repository checkout does not require credentials.

First, keep **DEPLOY_LOCAL=false** to validate the CI build safely. The pipeline has no server connection settings beyond the optional local deployment.

## Pipeline stages

1. Checkout from GitHub and verify tools/source.
2. Optional frontend checks with Node 22: `npm ci --force`, Vitest and `npm run check`.
3. Optional backend checks with Python 3.12: `compileall` and Ruff.
4. Build `cs-ai:jenkins-BUILD_NUMBER` from the local `Dockerfile`.
5. Run an ephemeral CS.AI container and require its Docker health check to become **healthy**. Then tag the tested image as `cs-ai:latest`.
6. Optionally deploy the tested image onto the **same Docker host used by Jenkins**.

## Parameters

| Parameter | Default | Description |
|---|---|---|
| `RUN_FRONTEND_TESTS` | `true` | Run frontend checks |
| `RUN_BACKEND_CHECKS` | `true` | Run backend checks |
| `SLIM_IMAGE` | `true` | Smaller image; some offline embedding, audio, and document processing features are unavailable |
| `DEPLOY_LOCAL` | `false` | Stop/replace a local Docker container named `cs-ai` |
| `APP_PORT` | `3000` | Port on the Jenkins Docker host when deployment is enabled |
| `OLLAMA_BASE_URL` | `http://host.docker.internal:11434` | Ollama endpoint the deployed container can reach |

## Enable local deployment

For an initial build, leave `DEPLOY_LOCAL=false`. Once the build passes, run **Build with Parameters** and set `DEPLOY_LOCAL=true` to run CS.AI on the Jenkins Docker host. Then visit `http://YOUR_JENKINS_DOCKER_HOST:3000` (or your chosen `APP_PORT`).

**Important:** Local deployment replaces an existing container named `cs-ai`. Persistent app data lives in the Docker named volume `cs-ai-data`; do not delete this volume if you need your accounts or chat data. The Docker host needs an available port and an accessible Ollama service (or you may configure OpenAI-compatible endpoints in the CS.AI admin UI). The default `OLLAMA_BASE_URL` assumes Ollama runs on the Docker host; change it if necessary.

This configuration does **not** publish to GHCR, Docker Hub, Kubernetes, or any remote server. Remote deployment requires server-specific credentials and configuration. Secrets must be stored in Jenkins Credentials, not committed to the repo.

## Optional automatic runs

You can enable a webhook in GitHub and an appropriate Jenkins GitHub integration, or configure a Jenkins **Poll SCM** schedule such as `H/5 * * * *` to check GitHub for changes. You may also trigger builds manually.

## Troubleshooting

- **`docker: command not found` / permission denied:** Jenkins must run on a Linux agent with working Docker access.
- **Frontend build runs out of memory:** Increase the Docker builder host's memory; the upstream build is resource-intensive.
- **Smoke test failed:** Open Jenkins Console Output and check the displayed container logs.
- **App starts but does not show models:** Confirm Ollama is running and `OLLAMA_BASE_URL` is reachable from the application container.
- **A CI check fails:** Read the specific lint/test output; do not bypass failed checks in order to deploy.

## Attribution

CS.AI is an unofficial personal hobby fork of [Open WebUI](https://github.com/open-webui/open-webui). The original `LICENSE` remains in place. Custom branding must comply with that license and any permissions applicable to use of the Herat University Computer Science emblem.
