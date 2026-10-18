# Test and Validation Plan

## Local application tests

```bash
python3 -m unittest discover -s tests -v
```

Expected: all three tests pass.

## Container validation

```bash
docker build -t cicd-demo:local .
docker run --rm -d --name cicd-demo-test -p 8081:8081 cicd-demo:local
curl -i http://localhost:8081/health
docker rm -f cicd-demo-test
```

Expected: `/health` returns HTTP 200 and `{\"status\": \"ok\"}`.

## Pipeline validation

| Case | Action | Expected result |
|---|---|---|
| Happy path | Push a valid commit | Test, image build/push, rollout and smoke check succeed |
| Unit-test failure | Temporarily add a failing assertion | Pipeline stops before image push/deployment |
| Missing image namespace | Leave `DOCKERHUB_NAMESPACE` empty | Configuration stage fails clearly |
| Registry authentication failure | Use a deliberately invalid test credential | Push stage fails without exposing token |
| Cluster/API unavailable | Use a test-only invalid kubeconfig | Deployment stage fails clearly |
| Rollout failure | Use a deliberately invalid image in a controlled test | Rollout times out and build fails |
| Repeat run | Run the same commit again | Deployment remains healthy |

Restore all temporary changes and credentials after negative tests. Never intentionally break a shared or production cluster.
