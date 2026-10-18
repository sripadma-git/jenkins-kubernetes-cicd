# Jenkins + GitHub + Docker + Kind CI/CD Assessment

A small, end-to-end CI/CD demonstration using GitHub, Jenkins Declarative Pipeline (Groovy), Docker Hub, and a local Kind Kubernetes cluster.

## Architecture

1. A developer pushes code to GitHub.
2. A GitHub webhook triggers the Jenkins Pipeline.
3. Jenkins checks out the commit and runs unit tests.
4. Jenkins builds a Docker image tagged with the Git commit SHA.
5. Jenkins pushes the image to Docker Hub.
6. Jenkins deploys the image to Kubernetes using `kubectl` and verifies the rollout.

> Security: Never commit credentials, Docker Hub tokens, or kubeconfig files to this repository. Store them in Jenkins Credentials. A local Kind kubeconfig often needs its API endpoint adjusted so Jenkins can reach it; follow `docs/SETUP.md`.

## Repository layout

```text
.
├── Jenkinsfile
├── Dockerfile
├── .dockerignore
├── .gitignore
├── requirements.txt
├── app/
│   └── app.py
├── tests/
│   └── test_app.py
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── docs/
    ├── SETUP.md
    ├── SECURITY.md
    └── TESTING.md
```

## Quick local test

Requires Python 3.11+.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python -m unittest discover -s tests -v
```

Build and run the container:

```bash
docker build -t cicd-demo:local .
docker run --rm -p 8081:8081 cicd-demo:local
# In another terminal:
curl -i http://localhost:8081/health
```

## Jenkins configuration summary

Create Jenkins credentials:
- `dockerhub-creds`: Username with password; use a Docker Hub access token as the password.
- `kubeconfig`: Secret file containing a kubeconfig that is reachable from the Jenkins agent.
- For a private GitHub repository, configure Git credentials as needed; public repositories can be checked out without credentials.

Set the Jenkins job to **Pipeline script from SCM**, select Git, set the repository URL and branch `*/main`, and use `Jenkinsfile` as the script path. Enable the GitHub push trigger and configure a GitHub webhook to `https://YOUR-REACHABLE-JENKINS-URL/github-webhook/`.

Set the following job parameters when running:
- `DOCKERHUB_NAMESPACE`: your Docker Hub username or organization.
- `IMAGE_NAME`: image repository name (default `jenkins-kubernetes-cicd-demo`).
- `K8S_NAMESPACE`: target namespace (default `cicd-demo`).
- `K8S_DEPLOYMENT`: Deployment name (default `cicd-demo`).
- `K8S_CONTAINER`: container name (default `app`).

The Docker Hub namespace must be supplied before the image stage. The pipeline intentionally fails early if it is missing.

## Important local Kind note

Kind nodes run as Docker containers. A kubeconfig exported on the host can contain an API endpoint such as `https://127.0.0.1:xxxxx`, which points to the Jenkins container itself when used inside Jenkins. Use a reachable API endpoint from the Jenkins agent network, and ensure the certificate authority data remains valid. See `docs/SETUP.md`. Do not blindly replace the API endpoint without considering TLS SANs and network routing.

## Limitations

This is an assessment/demo setup, not a production reference architecture. For production, use managed Kubernetes, least-privilege service accounts, protected branches, image scanning/signing, TLS-secured webhook delivery, and controlled promotion/approval policies.
