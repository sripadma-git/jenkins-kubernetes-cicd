pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    triggers {
        githubPush()
    }

    parameters {
        string(name: 'DOCKERHUB_NAMESPACE', defaultValue: '', description: 'Docker Hub username or organization (required)')
        string(name: 'IMAGE_NAME', defaultValue: 'jenkins-kubernetes-cicd-demo', description: 'Docker Hub repository/image name')
        string(name: 'K8S_NAMESPACE', defaultValue: 'cicd-demo', description: 'Kubernetes namespace')
        string(name: 'K8S_DEPLOYMENT', defaultValue: 'cicd-demo', description: 'Kubernetes Deployment name')
        string(name: 'K8S_CONTAINER', defaultValue: 'app', description: 'Container name in the Deployment')
    }

    environment {
        REGISTRY_CREDENTIALS = 'dockerhub-creds'
        KUBECONFIG_CREDENTIAL = 'kubeconfig'
        APP_PORT = '8081'
    }

    stages {
        stage('Validate Configuration') {
            steps {
                script {
                    if (!params.DOCKERHUB_NAMESPACE?.trim()) {
                        error('Set DOCKERHUB_NAMESPACE to your Docker Hub username or organization.')
                    }
                    if (!(params.IMAGE_NAME ==~ /[a-z0-9]+([._-][a-z0-9]+)*/)) {
                        error('IMAGE_NAME must contain lowercase letters, digits, dots, underscores, or hyphens.')
                    }
                    env.IMAGE_REPOSITORY = "${params.DOCKERHUB_NAMESPACE.trim()}/${params.IMAGE_NAME}"
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
                    env.GIT_SHA = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = env.GIT_SHA
                    env.FULL_IMAGE = "${env.IMAGE_REPOSITORY}:${env.IMAGE_TAG}"
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
                sh '''
                    set -eu
                    docker build --pull -t "$FULL_IMAGE" .
                '''
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
                        set +x
                        printf '%s' "$DOCKERHUB_TOKEN" | docker login \
                          --username "$DOCKERHUB_USER" --password-stdin
                        set -eu
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
                          -n "$K8S_NAMESPACE" --timeout=180s

                        kubectl get deployment,pods,services -n "$K8S_NAMESPACE"
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
                        kubectl get deployment "$K8S_DEPLOYMENT" -n "$K8S_NAMESPACE" \
                          -o jsonpath='{.status.availableReplicas}' | grep -E '^[1-9][0-9]*$'
                        echo "Deployment has at least one available replica."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "CI/CD succeeded: ${env.FULL_IMAGE ?: 'image not resolved'}"
        }
        failure {
            echo 'CI/CD failed. Review the first failed stage and its console output.'
        }
        always {
            echo "Build #${env.BUILD_NUMBER} result: ${currentBuild.currentResult}"
        }
    }
}
