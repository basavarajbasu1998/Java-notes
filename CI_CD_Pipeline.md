# CI/CD Pipeline

```
Developer                     GitHub                     CI (GitHub Actions/Jenkins)                    Environments
   │ git push feature branch     │                                │
   ├───────────────────────────► PR opened ─────────────────────► 1 build + unit tests
   │                             │  reviewers comment             2 static analysis (Sonar), dependency/security scan
   │                             │◄───────── status checks ───────3 integration tests (Testcontainers)
   │ merge to main ─────────────►│──────────────────────────────► 4 docker build + tag (git sha) → push ECR
                                                                  5 deploy to DEV (auto) → smoke tests
                                                                  6 deploy to STAGING → regression/perf tests
                                                                  7 manual approval → PROD (rolling/canary)
                                                                  8 monitor (CloudWatch/Grafana) → rollback if error rate up
```
GitHub Actions example:
```yaml
name: ci
on: { pull_request: { branches: [main] }, push: { branches: [main] } }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 21, cache: maven }
      - run: mvn -B verify
      - if: github.ref == 'refs/heads/main'
        run: |
          docker build -t $ECR/order-service:${{ github.sha }} .
          docker push $ECR/order-service:${{ github.sha }}
```
Deployment strategies: **rolling** (replace gradually), **blue-green** (two envs, switch), **canary** (small % first), **feature flags** (ship code dark, enable later). Config per environment via profiles/ConfigMaps; secrets via Secrets Manager. Database changes via Flyway, backward-compatible ("expand then contract").

**Observability trio:** logs (ELK/CloudWatch) · metrics (Prometheus/Grafana: latency, error rate, saturation) · traces (Jaeger/X-Ray). Alerts on symptoms (error rate, p99 latency), not just CPU.
