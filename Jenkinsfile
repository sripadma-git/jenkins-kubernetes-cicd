
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    parameters {
        string(
            name: 'DOCKERHUB_NAMESPACE',
            defaultValue: '',
            description: 'Docker Hub username or organization (required)'
        )
        string(
            name: 'IMAGE_NAME',
            defaultValue: 'jenkins-kubernetes-cicd-demo',
            description: 'Docker Hub repository/image name'
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
            description: 'Container name in the Deployment'
        )
    }

    environment {
        REGISTRY_CREDENTIALS = 'dockerhub-creds'
        KUBECONFIG_CREDENTIAL = 'kubeconfig'
        REGISTRY_HOST = 'docker.io'
        APP_PORT = '8081'
    }

    stages {
        stage('Validate Configuration') {
            steps {
                script {
                    env.DOCKERHUB_NAMESPACE =
                        params.DOCKERHUB_NAMESPACE.trim()
                    env.IMAGE_NAME = params.IMAGE_NAME.trim()
                    env.K8S_NAMESPACE = params.K8S_NAMESPACE.trim()
                    env.K8S_DEPLOYMENT = params.K8S_DEPLOYMENT.trim()
                    env.K8S_CONTAINER = params.K8S_CONTAINER.trim()

                    if (!env.DOCKERHUB_NAMESPACE ||
                        !env.IMAGE_NAME ||
                        !env.K8S_NAMESPACE ||
                        !env.K8S_DEPLOYMENT ||
                        !env.K8S_CONTAINER) {
                        error('Required build parameters cannot be empty.')
                    }

                    if (!(env.IMAGE_NAME ==~ /[a-z0-9]+([._-][a-z0-9]+)*/)) {
                        error('IMAGE_NAME contains invalid characters.')
                    }

                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    env.FULL_IMAGE =
                        "${env.DOCKERHUB_NAMESPACE}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    echo "Building ${env.FULL_IMAGE}"
                }

                sh '''
                    set -eu
                    command -v git
                    command -v docker
                    command -v kubectl
                    command -v python3
                '''
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
                        "${env.DOCKERHUB_NAMESPACE}/${env.IMAGE_NAME}:${env.IMAGE_TAG}"

                    echo "Building ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    set -eu
                    python3 -m unittest discover -s tests -v
                '''
            }
        }

        stage('Build Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${env.REGISTRY_CREDENTIALS}",
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set -eu
                        set +x

                        printf '%s' "$DOCKERHUB_TOKEN" | docker login \
                          --username "$DOCKERHUB_USER" --password-stdin

                        docker build --pull -t "$FULL_IMAGE" .
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${env.REGISTRY_CREDENTIALS}",
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        set -eu
                        set +x

                        printf '%s' "$DOCKERHUB_TOKEN" | docker login \
                          --username "$DOCKERHUB_USER" --password-stdin

                        docker push "$FULL_IMAGE"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: "${env.KUBECONFIG_CREDENTIAL}",
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl cluster-info
                        kubectl apply -f k8s/namespace.yaml
                        kubectl apply -n "$K8S_NAMESPACE" -f k8s/service.yaml
                        kubectl apply -n "$K8S_NAMESPACE" -f k8s/deployment.yaml

                        kubectl set image \
                          "deployment/$K8S_DEPLOYMENT" \
                          "$K8S_CONTAINER=$FULL_IMAGE" \
                          -n "$K8S_NAMESPACE"

                        kubectl rollout status \
                          "deployment/$K8S_DEPLOYMENT" \
                          -n "$K8S_NAMESPACE" \
                          --timeout=180s
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {
                withCredentials([file(
                    credentialsId: "${env.KUBECONFIG_CREDENTIAL}",
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl get deployment "$K8S_DEPLOYMENT" \
                          -n "$K8S_NAMESPACE" \
                          -o jsonpath='{.status.availableReplicas}' \
                          | grep -E '^[1-9][0-9]*$'

                        echo "Deployment has at least one available replica."
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

