// CS.AI — Jenkins Pipeline
// Source: https://github.com/miladKatebzadeh/CS-AI
// Job type: Pipeline -> Pipeline script from SCM (Git), branch */main, script path Jenkinsfile
// Agent: Linux with Git and a working Docker Engine/CLI. Docker daemon access is required.
// This pipeline builds locally; it does NOT push images to a registry.
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 150, unit: 'MINUTES')
    }

    parameters {
        booleanParam(name: 'RUN_FRONTEND_TESTS', defaultValue: true, description: 'Install Node 22 dependencies; run Vitest and Svelte checks.')
        booleanParam(name: 'RUN_BACKEND_CHECKS', defaultValue: true, description: 'Compile Python and run Ruff formatting/lint checks.')
        booleanParam(name: 'SLIM_IMAGE', defaultValue: true, description: 'Build the smaller image (some offline audio/document/model features are omitted).')
        booleanParam(name: 'DEPLOY_LOCAL', defaultValue: false, description: 'Replace the CS.AI container on the Jenkins Docker host after a successful build and smoke test.')
        string(name: 'APP_PORT', defaultValue: '3000', description: 'Host port for optional local deployment.')
        string(name: 'OLLAMA_BASE_URL', defaultValue: 'http://host.docker.internal:11434', description: 'Ollama URL reachable from the deployed container.')
    }

    environment {
        IMAGE_NAME = 'cs-ai'
        CONTAINER_NAME = 'cs-ai'
        DATA_VOLUME = 'cs-ai-data'
        DOCKER_BUILDKIT = '1'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git rev-parse --short=12 HEAD'
            }
        }

        stage('Prerequisites') {
            steps {
                sh '''#!/bin/sh
set -eu
command -v git >/dev/null
command -v docker >/dev/null
docker info >/dev/null
test -f Dockerfile
test -f package-lock.json
test -f backend/open_webui/env.py
grep -q 'CS.AI' src/app.html
echo "Docker is available; CS.AI sources checked."
'''
            }
        }

        stage('Frontend tests') {
            when { expression { return params.RUN_FRONTEND_TESTS } }
            steps {
                sh '''#!/bin/sh
set -eu
docker run --rm \
    --user "$(id -u):$(id -g)" \
    -e HOME=/tmp \
    -e npm_config_cache=/tmp/.npm \
    -e ONNXRUNTIME_NODE_INSTALL_CUDA=skip \
    -e NODE_OPTIONS=--max-old-space-size=8192 \
    -v "$PWD:/workspace" \
    -w /workspace \
    node:22-bookworm \
    sh -ec 'npm ci --force && npm run test:frontend && npm run check'
'''
            }
        }

        stage('Backend checks') {
            when { expression { return params.RUN_BACKEND_CHECKS } }
            steps {
                sh '''#!/bin/sh
set -eu
docker run --rm \
    -v "$PWD:/workspace" \
    -w /workspace \
    python:3.12-slim \
    sh -ec '
        pip install --quiet --no-cache-dir ruff==0.15.5
        python -m compileall -q backend/open_webui
        ruff format --check . --exclude .venv --exclude venv
        ruff check --select=F --ignore=F401,F403,F405,F541,F811,F841 .
    '
'''
            }
        }

        stage('Docker build') {
            steps {
                sh '''#!/bin/sh
set -eu
docker build \
    --build-arg "BUILD_HASH=$(git rev-parse HEAD)" \
    --build-arg "BUILD_CHANNEL=jenkins" \
    --build-arg "USE_SLIM=$SLIM_IMAGE" \
    -t "$IMAGE_NAME:jenkins-$BUILD_NUMBER" \
    .
'''
            }
        }

        stage('Container smoke test') {
            steps {
                sh '''#!/bin/sh
set -eu
smoke="cs-ai-smoke-$BUILD_NUMBER"
docker rm -f "$smoke" >/dev/null 2>&1 || true
docker run -d --name "$smoke" \
    -e WEBUI_NAME=CS.AI \
    -e WEBUI_AUTH=false \
    -e OFFLINE_MODE=true \
    "$IMAGE_NAME:jenkins-$BUILD_NUMBER" >/dev/null

# The Dockerfile provides an HTTP /health healthcheck.
ok=0
i=0
while [ "$i" -lt 48 ]; do
    status="$(docker inspect --format '{{if .State.Health}}{{.State.Health.Status}}{{else}}missing{{end}}' "$smoke")"
    if [ "$status" = healthy ]; then
        ok=1
        break
    fi
    if [ "$status" = missing ]; then
        echo 'Image is missing the expected Docker healthcheck.' >&2
        break
    fi
    i=$((i + 1))
    sleep 5
done

if [ "$ok" -ne 1 ]; then
    docker ps -a --filter "name=$smoke"
    docker logs --tail=120 "$smoke" || true
    echo 'CS.AI container did not pass its health check.' >&2
    exit 1
fi
docker tag "$IMAGE_NAME:jenkins-$BUILD_NUMBER" "$IMAGE_NAME:latest"
echo "CS.AI Docker image ready: $IMAGE_NAME:latest"
'''
            }
        }

        stage('Deploy on Jenkins Docker host') {
            when { expression { return params.DEPLOY_LOCAL } }
            steps {
                sh '''#!/bin/sh
set -eu
case "$APP_PORT" in
  ''|*[!0-9]*) echo 'APP_PORT must be numeric' >&2; exit 1 ;;
esac
if [ "$APP_PORT" -lt 1 ] || [ "$APP_PORT" -gt 65535 ]; then
    echo 'APP_PORT must be between 1 and 65535' >&2
    exit 1
fi
docker volume create "$DATA_VOLUME" >/dev/null
docker rm -f "$CONTAINER_NAME" >/dev/null 2>&1 || true
docker run -d \
    --name "$CONTAINER_NAME" \
    --restart unless-stopped \
    -p "$APP_PORT:8080" \
    --add-host host.docker.internal:host-gateway \
    -e WEBUI_NAME=CS.AI \
    -e OLLAMA_BASE_URL="$OLLAMA_BASE_URL" \
    -v "$DATA_VOLUME:/app/backend/data" \
    "$IMAGE_NAME:latest"
echo "CS.AI container started on port $APP_PORT."
'''
            }
        }
    }

    post {
        always {
            sh 'docker rm -f "cs-ai-smoke-$BUILD_NUMBER" >/dev/null 2>&1 || true'
        }
        success {
            echo 'CS.AI Jenkins pipeline finished successfully.'
        }
        failure {
            echo 'CS.AI Jenkins pipeline failed; inspect the failing stage and console logs.'
        }
    }
}
