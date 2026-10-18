
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
    }

    parameters {
        string(
            name: 'DOCKERHUB_NAMESPACE',
            defaultValue: '',
            description: 'Docker Hub username or organization, e.g. sripadma996'
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
            description: 'Kubernetes Deployment name'
        )
        string(
            name: 'K8S_CONTAINER',
            defaultValue: 'app',
            description: 'Container name inside the Deployment'
        )
    }

    environment {
        REGISTRY_HOST = 'docker.io'
        DOCKER_CREDENTIALS_ID = 'dockerhub-creds'
        KUBECONFIG_CREDENTIAL = 'kubeconfig'
    }

    stages {
        stage('Validate Configuration') {
            steps {
                script {
                    if (!params.DOCKERHUB_NAMESPACE?.trim()) {
                        error(
                            'DOCKERHUB_NAMESPACE is required. ' +
                            'Set it to your Docker Hub username.'
                        )
                    }

                    if (!(params.IMAGE_NAME ==~ /[a-zA-Z0-9._-]+/)) {
                        error('IMAGE_NAME contains invalid characters.')
                    }

                    if (!(params.K8S_NAMESPACE ==~ /[a-z0-9]([-a-z0-9]*[a-z0-9])?/)) {
                        error('K8S_NAMESPACE is not a valid Kubernetes namespace.')
                    }

                    if (!(params.K8S_DEPLOYMENT ==~ /[a-z0-9]([-a-z0-9]*[a-z0-9])?/)) {
                        error('K8S_DEPLOYMENT is not a valid Kubernetes resource name.')
                    }

                    if (!(params.K8S_CONTAINER ==~ /[a-z0-9]([-a-z0-9]*[a-z0-9])?/)) {
                        error('K8S_CONTAINER is not a valid container name.')
                    }

                    env.IMAGE_REPOSITORY =
                        "${params.DOCKERHUB_NAMESPACE.trim()}/${params.IMAGE_NAME.trim()}"

                    env.IMAGE_TAG = env.GIT_COMMIT.take(12)

                    env.FULL_IMAGE =
                        "${env.REGISTRY_HOST}/${env.IMAGE_REPOSITORY}:${env.IMAGE_TAG}"

                    echo "Image repository: ${env.IMAGE_REPOSITORY}"
                    echo "Image tag: ${env.IMAGE_TAG}"
                    echo "Kubernetes namespace: ${params.K8S_NAMESPACE}"
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm

                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    env.FULL_IMAGE =
                        "${env.REGISTRY_HOST}/${env.IMAGE_REPOSITORY}:${env.IMAGE_TAG}"

                    echo "Checked-out commit: ${env.IMAGE_TAG}"
                    echo "Image to build: ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    set -eu

                    echo "Running Python unit tests..."
                    python3 -m unittest discover \
                      -s app \
                      -p 'test_*.py' \
                      -v
                '''
            }
        }

        stage('Build Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${env.DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -eu

                        echo "Authenticating to Docker Hub..."
                        printf '%s' "$DOCKER_PASSWORD" |
                          docker login "$REGISTRY_HOST" \
                            --username "$DOCKER_USER" \
                            --password-stdin

                        echo "Building image: $FULL_IMAGE"
                        docker build --pull --tag "$FULL_IMAGE" .

                        docker logout "$REGISTRY_HOST" || true
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${env.DOCKER_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -eu

                        echo "Authenticating to Docker Hub..."
                        printf '%s' "$DOCKER_PASSWORD" |
                          docker login "$REGISTRY_HOST" \
                            --username "$DOCKER_USER" \
                            --password-stdin

                        echo "Pushing image: $FULL_IMAGE"
                        docker push "$FULL_IMAGE"

                        docker logout "$REGISTRY_HOST" || true
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: "${env.KUBECONFIG_CREDENTIAL}",
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        echo "Checking Kubernetes connectivity..."
                        kubectl cluster-info

                        echo "Applying Kubernetes manifests..."
                        kubectl apply -f k8s/namespace.yaml
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml

                        echo "Updating deployment image..."
                        kubectl set image \
                          "deployment/$K8S_DEPLOYMENT" \
                          "$K8S_CONTAINER=$FULL_IMAGE" \
                          -n "$K8S_NAMESPACE"

                        echo "Waiting for rollout..."
                        kubectl rollout status \
                          "deployment/$K8S_DEPLOYMENT" \
                          -n "$K8S_NAMESPACE" \
                          --timeout=180s

                        echo "Deployment status:"
                        kubectl get deployment,service \
                          -n "$K8S_NAMESPACE"

                        kubectl get pods -n "$K8S_NAMESPACE" -o wide
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {
                withCredentials([
                    file(
                        credentialsId: "${env.KUBECONFIG_CREDENTIAL}",
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        PORT=18081
                        RESPONSE_FILE="health-response-${BUILD_NUMBER}.json"
                        PF_LOG="port-forward-${BUILD_NUMBER}.log"
                        PF_PID=""

                        cleanup() {
                            if [ -n "$PF_PID" ]; then
                                kill "$PF_PID" 2>/dev/null || true
                                wait "$PF_PID" 2>/dev/null || true
                            fi

                            rm -f "$RESPONSE_FILE" "$PF_LOG"
                        }

                        trap cleanup EXIT

                        echo "Checking deployment rollout..."
                        kubectl rollout status \
                          "deployment/$K8S_DEPLOYMENT" \
                          -n "$K8S_NAMESPACE" \
                          --timeout=120s

                        echo "Checking replica availability..."
                        DESIRED=$(kubectl get deployment "$K8S_DEPLOYMENT" \
                          -n "$K8S_NAMESPACE" \
                          -o jsonpath='{.spec.replicas}')

                        AVAILABLE=$(kubectl get deployment "$K8S_DEPLOYMENT" \
                          -n "$K8S_NAMESPACE" \
                          -o jsonpath='{.status.availableReplicas}')

                        DESIRED=${DESIRED:-0}
                        AVAILABLE=${AVAILABLE:-0}

                        if [ "$DESIRED" -lt 1 ] ||
                           [ "$AVAILABLE" -lt "$DESIRED" ]; then
                            echo "ERROR: Expected $DESIRED available replicas; found $AVAILABLE."
                            exit 1
                        fi

                        echo "Available replicas: $AVAILABLE/$DESIRED"

                        echo "Starting temporary Service port-forward..."
                        kubectl port-forward \
                          -n "$K8S_NAMESPACE" \
                          "service/$K8S_DEPLOYMENT" \
                          "$PORT:80" \
                          > "$PF_LOG" 2>&1 &
                        PF_PID=$!

                        echo "Waiting for HTTP health endpoint..."
                        READY=0

                        for i in $(seq 1 30); do
                            if ! kill -0 "$PF_PID" 2>/dev/null; then
                                cat "$PF_LOG"
                                echo "ERROR: Port-forward process stopped."
                                exit 1
                            fi

                            if curl --silent --show-error \
                                --fail --max-time 3 \
                                "http://127.0.0.1:$PORT/health" \
                                -o "$RESPONSE_FILE" 2>/dev/null; then
                                READY=1
                                break
                            fi

                            sleep 2
                        done

                        if [ "$READY" -ne 1 ]; then
                            cat "$PF_LOG"
                            echo "ERROR: /health did not return HTTP 200."
                            exit 1
                        fi

                        echo "Health endpoint response:"
                        cat "$RESPONSE_FILE"
                        echo

                        if ! grep -Eq \
                          '"status"[[:space:]]*:[[:space:]]*"ok"' \
                          "$RESPONSE_FILE"; then
                            echo "ERROR: Health response did not contain status=ok."
                            exit 1
                        fi

                        echo "Smoke Test PASSED:"
                        echo "- Deployment rollout successful"
                        echo "- All desired replicas available"
                        echo "- /health returned HTTP 200"
                        echo "- Response contains status=ok"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD pipeline completed successfully: ${env.FULL_IMAGE}"
        }

        failure {
            echo "Build #${env.BUILD_NUMBER} failed. Review the first failed stage and its console output."
        }

        always {
            echo "Build #${env.BUILD_NUMBER} result: ${currentBuild.currentResult}"
        }
    }
}

