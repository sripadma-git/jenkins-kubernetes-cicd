
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
        skipDefaultCheckout(true)
    }

    parameters {
        string(
            name: 'DOCKERHUB_NAMESPACE',
            defaultValue: 'sripadma996',
            description: 'Docker Hub username or namespace'
        )

        string(
            name: 'IMAGE_NAME',
            defaultValue: 'jenkins-kubernetes-cicd-demo',
            description: 'Docker image repository name'
        )

        string(
            name: 'K8S_NAMESPACE',
            defaultValue: 'cicd-demo',
            description: 'Kubernetes namespace'
        )

        string(
            name: 'K8S_DEPLOYMENT',
            defaultValue: 'cicd-demo',
            description: 'Kubernetes deployment name'
        )

        string(
            name: 'K8S_CONTAINER',
            defaultValue: 'app',
            description: 'Container name in the deployment'
        )
    }

    environment {
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        KUBECONFIG_CREDENTIALS_ID = 'kubeconfig'
        LOCAL_HEALTH_PORT = '18081'
    }

    stages {
        stage('Validate Configuration') {
            steps {
                script {
                    def requiredValues = [
                        'DOCKERHUB_NAMESPACE': params.DOCKERHUB_NAMESPACE,
                        'IMAGE_NAME': params.IMAGE_NAME,
                        'K8S_NAMESPACE': params.K8S_NAMESPACE,
                        'K8S_DEPLOYMENT': params.K8S_DEPLOYMENT,
                        'K8S_CONTAINER': params.K8S_CONTAINER
                    ]

                    requiredValues.each { key, value ->
                        if (!value?.trim()) {
                            error("${key} must not be empty")
                        }

                        if (!(value ==~ /[a-z0-9]([a-z0-9._-]*[a-z0-9])?/)) {
                            error("${key} contains invalid characters: ${value}")
                        }
                    }

                    if (params.DOCKERHUB_NAMESPACE.contains('/')) {
                        error('DOCKERHUB_NAMESPACE must not contain a slash')
                    }

                    echo 'Configuration validation passed.'
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_REPOSITORY =
                        "${params.DOCKERHUB_NAMESPACE}/${params.IMAGE_NAME}"

                    env.IMAGE_TAG = env.GIT_COMMIT_SHORT
                    env.FULL_IMAGE =
                        "docker.io/${env.IMAGE_REPOSITORY}:${env.IMAGE_TAG}"

                    echo "Source commit: ${env.GIT_COMMIT_SHORT}"
                    echo "Image to build: ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Unit Tests') {
            steps {
                sh(
                    label: 'Run Python unit tests',
                    script: '''#!/bin/sh
set -eu

if [ ! -d tests ]; then
    echo "ERROR: tests directory was not found."
    exit 1
fi

if ! find tests -type f -name 'test_*.py' -print -quit | grep -q .; then
    echo "ERROR: No test_*.py files found under tests/."
    exit 1
fi

set +e
python3 -m unittest discover -s tests -p 'test_*.py' -v > unittest-output.txt 2>&1
test_status=$?
set -e

cat unittest-output.txt

if [ "$test_status" -ne 0 ]; then
    echo "ERROR: One or more unit tests failed."
    exit "$test_status"
fi

if ! grep -Eq 'Ran [1-9][0-9]* tests?' unittest-output.txt; then
    echo "ERROR: Zero unit tests were executed."
    exit 1
fi

echo "Unit tests passed."
'''
                )
            }
            post {
                always {
                    archiveArtifacts(
                        artifacts: 'unittest-output.txt',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Build Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh(
                        label: 'Build Docker image',
                        script: '''#!/bin/sh
set -eu

printf '%s' "$DOCKER_PASSWORD" |
    docker login -u "$DOCKER_USERNAME" --password-stdin

docker build --pull --tag "$FULL_IMAGE" .
'''
                    )
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh(
                        label: 'Push Docker image',
                        script: '''#!/bin/sh
set -eu

printf '%s' "$DOCKER_PASSWORD" |
    docker login -u "$DOCKER_USERNAME" --password-stdin

docker push "$FULL_IMAGE"
'''
                    )
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: "${KUBECONFIG_CREDENTIALS_ID}",
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh(
                        label: 'Deploy application',
                        script: '''#!/bin/sh
set -eu

kubectl --kubeconfig "$KUBECONFIG" apply -f k8s/namespace.yaml
kubectl --kubeconfig "$KUBECONFIG" apply -n "$K8S_NAMESPACE" -f k8s/deployment.yaml
kubectl --kubeconfig "$KUBECONFIG" apply -n "$K8S_NAMESPACE" -f k8s/service.yaml

kubectl --kubeconfig "$KUBECONFIG" \
    set image "deployment/$K8S_DEPLOYMENT" \
    "$K8S_CONTAINER=$FULL_IMAGE" \
    -n "$K8S_NAMESPACE"

kubectl --kubeconfig "$KUBECONFIG" \
    rollout status "deployment/$K8S_DEPLOYMENT" \
    -n "$K8S_NAMESPACE" \
    --timeout=180s

kubectl --kubeconfig "$KUBECONFIG" \
    get deployment "$K8S_DEPLOYMENT" \
    -n "$K8S_NAMESPACE"
'''
                    )
                }
            }
        }

        stage('Smoke Test') {
            steps {
                withCredentials([
                    file(
                        credentialsId: "${KUBECONFIG_CREDENTIALS_ID}",
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh(
                        label: 'Verify deployment and HTTP health endpoint',
                        script: '''#!/bin/sh
set -eu

PORT="$LOCAL_HEALTH_PORT"
RESPONSE_FILE="$(mktemp)"
PF_LOG="$(mktemp)"
PF_PID=""

cleanup() {
    if [ -n "$PF_PID" ]; then
        kill "$PF_PID" 2>/dev/null || true
        wait "$PF_PID" 2>/dev/null || true
    fi

    rm -f "$RESPONSE_FILE" "$PF_LOG"
}

trap cleanup EXIT INT TERM

kubectl --kubeconfig "$KUBECONFIG" \
    rollout status "deployment/$K8S_DEPLOYMENT" \
    -n "$K8S_NAMESPACE" \
    --timeout=180s

DESIRED="$(kubectl --kubeconfig "$KUBECONFIG" \
    get deployment "$K8S_DEPLOYMENT" \
    -n "$K8S_NAMESPACE" \
    -o jsonpath='{.spec.replicas}')"

AVAILABLE="$(kubectl --kubeconfig "$KUBECONFIG" \
    get deployment "$K8S_DEPLOYMENT" \
    -n "$K8S_NAMESPACE" \
    -o jsonpath='{.status.availableReplicas}')"

DESIRED="${DESIRED:-0}"
AVAILABLE="${AVAILABLE:-0}"

echo "Desired replicas: $DESIRED"
echo "Available replicas: $AVAILABLE"

if [ "$DESIRED" -lt 1 ] || [ "$AVAILABLE" -lt "$DESIRED" ]; then
    echo "ERROR: Not all desired replicas are available."
    exit 1
fi

kubectl --kubeconfig "$KUBECONFIG" \
    port-forward \
    -n "$K8S_NAMESPACE" \
    "service/$K8S_DEPLOYMENT" \
    "$PORT:80" >"$PF_LOG" 2>&1 &

PF_PID=$!

HEALTH_OK=0

for attempt in $(seq 1 30); do
    if curl --silent --show-error --fail \
        --max-time 3 \
        "http://127.0.0.1:$PORT/health" \
        -o "$RESPONSE_FILE" 2>/dev/null; then
        HEALTH_OK=1
        break
    fi

    if ! kill -0 "$PF_PID" 2>/dev/null; then
        echo "ERROR: kubectl port-forward stopped unexpectedly."
        cat "$PF_LOG"
        exit 1
    fi

    sleep 2
done

if [ "$HEALTH_OK" -ne 1 ]; then
    echo "ERROR: The application health endpoint did not respond."
    cat "$PF_LOG"
    exit 1
fi

echo "Health endpoint response:"
cat "$RESPONSE_FILE"
echo

if ! grep -Eq '"status"[[:space:]]*:[[:space:]]*"ok"' "$RESPONSE_FILE"; then
    echo "ERROR: Health endpoint returned an unexpected response."
    exit 1
fi

echo "Smoke test passed: deployment is available and /health returned status ok."
'''
                    )
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline completed successfully."
            echo "Deployed image: ${env.FULL_IMAGE}"
        }

        failure {
            echo 'CI/CD pipeline failed. Review the stage logs above.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
