# Self-Healing-CICD-Pipeline-V2
# Self-Healing CI/CD v2: Progressive Delivery, Identity-Based Access and IaC

A deployment pipeline that builds one image, scans it, and deploys that exact image by digest to AWS using rolling, blue-green and canary strategies. Access uses short-lived identity instead of stored keys, and the whole environment is rebuilt from code. Every feature is proven by breaking it on purpose.

> v1 ([Self-Healing-CICD-Pipeline](https://github.com/RayyanSameer/Self-Healing-CICD-Pipeline)) proved build, scan, deploy, health check and automatic rollback. v2 closes every known v1 gap.

**Definition of "working in production":** `/health` returns 200 through nginx, and the digest of the running container equals the digest that passed the Trivy scan.

## Status

Legend: ✅ proven, 🚧 in progress, ⬜ not started. A feature is only ✅ after I broke it deliberately and saw the expected behaviour. Update as work lands.

| Phase | Feature | Proof of done | Est. | Status |
|---|---|---|---|---|
| 0 | Walking skeleton: build, push to GHCR, print digest | Green run, digest in logs | 1h | ⬜ |
| 1 | gunicorn, scan by digest, deploy by digest, container restart policy | Kill the container, it returns; running digest == scanned digest | 3h | ⬜ |
| 2 | nginx front door + **rolling** deploy | Broken version halts after the first replica; others keep serving | 3h | ⬜ |
| 3 | **Blue-green** + deeper `/health` (dependencies) | Broken new set never gets traffic; flip-back in seconds | 3h | ⬜ |
| 4 | **Canary** with metric-driven abort (nginx 5xx ratio) | Broken canary auto-aborted; majority of traffic unaffected | 3h | ⬜ |
| 5 | Identity and access: GitHub OIDC role, SSM Session Manager, port 22 closed, instance resolved by tag | No SSH key in GitHub secrets; deploy survives instance stop/start with no secret edits | 4h | ⬜ |
| 6 | Environments: `staging` + `production`, same digest promoted, smoke tests, manual approval, branch protection, `concurrency:` | Failing staging smoke test blocks production; two quick pushes do not race | 3h | ⬜ |
| 7 | Secrets in SSM Parameter Store, read via instance role | App starts from the store; removing the permission makes it fail visibly | 2h | ⬜ |
| 8 | Infrastructure as code: Terraform (instance, role, security group, SSM) | `destroy` then rebuild everything with one command | 4h | ⬜ |
| 9 | Maintenance automation: Dependabot/Renovate, scheduled rescans, Slack/email notifications, shellcheck stage, actions pinned to SHAs | Vulnerable pinned package produces an auto-PR with green checks; failed deploy sends a notification | 3h | ⬜ |
| 10 | Docs: README polish, architecture diagram, Loom walkthrough, error log | Someone can follow the README cold | 2h | ⬜ |

**Total estimate: about 31h. With a 30% buffer for debugging: about 40h.** Checkpoints: after Phase 4 (progressive delivery done, about 13h) and after Phase 6 (about 23h) the project is already a complete story if I must stop.

## v1 gaps and where v2 closes them

| v1 limitation | Closed in |
|---|---|
| Scanned image was not the deployed artifact | Phase 1 (digest) |
| Flask dev server (Werkzeug) in production | Phase 1 (gunicorn) |
| `nohup` as process supervisor | Phase 1 (restart policy) |
| All-or-nothing deploy, single version | Phases 2 to 4 |
| Shallow health check | Phase 3 |
| Manual `EC2_HOST` updates | Phase 5 (resolve by tag) |
| Long-lived SSH key, open SSH path | Phase 5 (OIDC + SSM) |
| Single environment | Phase 6 |
| No concurrency control | Phase 6 |
| Secrets handled outside a store | Phase 7 |
| Infrastructure created by hand | Phase 8 |
| Actions pinned to tags, no shellcheck, no notifications | Phase 9 |

## Architecture

```
GitHub Actions (OIDC role, no stored keys)
  build -> push to GHCR -> Trivy scans digest sha256:X
        |
        v  deploy by digest via SSM
EC2 (t3.micro, Terraform-managed)
  nginx :80
    |-- app replicas / blue / green / canary (gunicorn containers)
  config and secrets read from SSM Parameter Store via instance role
```

nginx is the single front door. Each strategy changes what nginx sends traffic to.

## Strategies

**Rolling:** replace replicas one at a time, waiting for a passing health check before the next. Failure stops the rollout with the remaining replicas still on the old version.

**Blue-green:** run a complete new set beside the current one; flip the nginx upstream only after the new set is healthy. Rollback is flipping back.

**Canary:** route a small weighted share to the new version, read the 5xx ratio from the nginx access log over a window, then promote or abort.

## Repo layout (planned)

```
basic_app/            Flask app + tests
Dockerfile            gunicorn, one worker per container
nginx/                upstream templates
scripts/              deploy scripts per strategy
terraform/            infrastructure
.github/workflows/    pipeline
```

## Run locally

_To fill in once Phase 1 works._

## Cost and constraints

- t3.micro (1 GB RAM): one gunicorn worker per container, small replica counts.
- No ALB, NAT gateway or Elastic IP. nginx on the instance handles traffic control.
- Staging is a second container set on the same instance (cheap, weaker isolation; documented trade-off).
- Instance stopped between sessions; Terraform `destroy` when idle.

## Errors and lessons

Recorded as they happen.

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| | | | |

## Stretch (only if time allows)

Reusable pipeline engine (after two real pipelines exist), Route 53 stable hostname, Prometheus-based canary analysis.
