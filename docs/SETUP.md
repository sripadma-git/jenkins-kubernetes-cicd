# Setup Guide

## 1. Prerequisites

- Windows with WSL2 Ubuntu, or a Linux host.
- Git, Docker, `kubectl`, and Kind installed.
- Docker Desktop running with WSL integration enabled if using WSL.
- A Jenkins controller and build agent with Git, Docker CLI/daemon access, Python 3, and `kubectl`.
- A GitHub repository and Docker Hub account.

Check tools:

```bash
git --version
docker info
kind version
kubectl version --client
python3 --version
```

## 2. Create a local Kind cluster

First inspect existing clusters:

```bash
kind get clusters
```

Delete only the cluster dedicated to this assessment if you are certain it is disposable:

```bash
kind delete cluster --name cicd-demo
```

Create the cluster:

```bash
cat > kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: cicd-demo
nodes:
  - role: control-plane
EOF

kind create cluster --config kind-config.yaml
kubectl cluster-info --context kind-cicd-demo
kubectl get nodes
```

Keep `kind-config.yaml` as local setup material if desired; do not commit credentials or exported kubeconfig files.

## 3. Test the app and image locally

From the repository root:

```bash
python3 -m unittest discover -s tests -v
docker build -t cicd-demo:local .
docker run --rm -p 8081:8081 cicd-demo:local
```

In another terminal:

```bash
curl -i http://localhost:8081/health
```

Stop the container with Ctrl+C.

## 4. Jenkins prerequisites and connectivity

The Jenkins agent that executes the pipeline needs:
- Git
- Python 3
- Docker CLI connected to a Docker daemon
- `kubectl`
- Network access to the Kubernetes API and Docker Hub

If Jenkins runs in a container, mounting the Docker socket allows access to the host daemon but is a powerful privilege. Use only in a disposable local assessment environment; avoid this pattern for untrusted workloads or production.

### Kind kubeconfig warning

Export the context for the Kind cluster:

```bash
kind get kubeconfig --name cicd-demo > kind-kubeconfig.yaml
```

Treat this file as a secret. Never commit it.

The exported kubeconfig typically uses a loopback API endpoint. From inside a Jenkins container, `127.0.0.1` refers to the Jenkins container, not the WSL host or Kind control-plane. The kubeconfig must use a Kubernetes API address reachable from the Jenkins agent. The API server certificate must also validate the chosen address.

Recommended local approach:
1. Decide whether Jenkins will run directly on the WSL host or inside a container.
2. Ensure Jenkins can reach the Kind API endpoint from that same network namespace.
3. Configure a stable, reachable API endpoint and appropriate TLS SANs in the Kind cluster configuration if needed.
4. Verify from the Jenkins agent with `kubectl --kubeconfig=/path/to/test-kubeconfig cluster-info` before storing the final file as a Jenkins Secret file credential.

Do not solve this by disabling TLS verification. Do not assume a kubeconfig that works from the host will work inside a container.

## 5. Add Jenkins credentials

In Manage Jenkins → Credentials → System → Global credentials, add:

- ID `dockerhub-creds`, kind **Username with password**. Username is the Docker Hub username; password is a Docker Hub access token.
- ID `kubeconfig`, kind **Secret file**. Upload the validated kubeconfig file reachable from the Jenkins agent.

Do not store credentials in the repository or hardcode them in the Jenkinsfile.

## 6. Create the Jenkins Pipeline job

1. Install/verify Pipeline, Git, GitHub, and Credentials Binding plugins.
2. Create a new Pipeline job.
3. Under Pipeline, select **Pipeline script from SCM**.
4. Select Git and enter the repository URL.
5. Set branch specifier to `*/main`.
6. Set Script Path to `Jenkinsfile`.
7. Save.
8. Run once manually with `DOCKERHUB_NAMESPACE` set to your Docker Hub namespace.

The pipeline needs `DOCKERHUB_NAMESPACE`, `IMAGE_NAME`, `K8S_NAMESPACE`, `K8S_DEPLOYMENT`, and `K8S_CONTAINER` parameters. Keep the Kubernetes names aligned with the manifests.

## 7. GitHub webhook

The Jenkins endpoint must be reachable by GitHub. In GitHub repository → Settings → Webhooks → Add webhook:
- Payload URL: `https://YOUR-REACHABLE-JENKINS-HOST/github-webhook/`
- Content type: `application/json`
- Event: push events

Enable the GitHub push trigger in the Jenkins job. A Jenkins instance available only at localhost cannot receive GitHub webhooks. For local development, use a secure temporary tunnel or a properly secured reachable endpoint; remove temporary exposure afterward.

## 8. Deploy and inspect

The Jenkins pipeline applies the namespace, Service and Deployment, updates the Deployment image to the commit-specific Docker Hub image, and waits for rollout completion.

Useful commands:

```bash
kubectl get all -n cicd-demo
kubectl rollout status deployment/cicd-demo -n cicd-demo
kubectl logs deployment/cicd-demo -n cicd-demo
kubectl port-forward service/cicd-demo 8081:80 -n cicd-demo
```

Then access `http://localhost:8081/health`.
