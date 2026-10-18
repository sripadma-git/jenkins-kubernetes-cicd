# Security Notes

- Never commit kubeconfig files, access tokens, passwords, or private keys.
- Store registry credentials and kubeconfig in Jenkins Credentials.
- Use a Docker Hub access token rather than an account password.
- Limit kubeconfig permissions to the namespace and resources required by the pipeline; do not use a cluster-admin credential in a production environment.
- Protect the main branch and require review for Jenkinsfile and deployment manifest changes.
- Do not disable Kubernetes TLS verification to work around a connectivity problem.
- Do not print secret values in logs. Avoid shell tracing around secret handling.
- Treat Docker socket access from Jenkins as highly privileged; use only for a trusted, isolated assessment environment.
- Consider image scanning, signed images, dependency pinning, and an isolated build agent before production use.
- A GitHub webhook endpoint must use HTTPS and appropriate access controls.
