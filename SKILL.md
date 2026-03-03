---
name: deployment-sorcerer
version: 2.1.0
description: Automated deployment and infrastructure orchestration with zero-config rollout strategies
author: SMOUJBOT
tags: [deployment, infrastructure, automation, cloud, orchestration]
dependencies:
  - terraform >= 1.5.0
  - kubectl >= 1.28.0
  - docker >= 24.0.0
  - ansible >= 2.15.0
  - jq >= 1.6
  - yq >= 4.30.0
required_env:
  - DEPLOYMENT_SORCERER_MODE (values: dry-run, staging, production)
  - CLOUD_PROVIDER (values: aws, gcp, azure, docker)
  - INFRASTRUCTURE_REPO (git repository URL)
  - APP_REPO (git repository URL)
  - ENVIRONMENT (values: dev, staging, prod)
optional_env:
  - TF_VAR_environment
  - K8S_NAMESPACE
  - DOCKER_REGISTRY
  - DEPLOY_WEBHOOK_SECRET
---

# Deployment Sorcerer

## Purpose

Deployment Sorcerer automates complete application infrastructure deployment with zero manual configuration. It handles multi-cloud infrastructure provisioning, container orchestration, blue-green rollouts, canary deployments, automated rollback on failure, and health verification—all from a single declarative configuration.

Real use cases:
- Deploy microservices to EKS/GKE/AKS with automatic service mesh injection (Istio/Linkerd)
- Execute zero-downtime blue-green deployments for monoliths on AWS ECS or App Runner
- Spin up complete dev environments with Terraform + Docker Compose in under 5 minutes
- Perform automated rollback when health checks fail or error rate exceeds threshold
- Manage infrastructure drift detection and auto-remediation
- Deploy to multiple regions with traffic shifting using Route53/Cloudflare
- Roll out database migrations with backup validation and automatic rollback on schema errors

## Scope

Deployment Sorcerer operates on:

**Infrastructure as Code:**
- Terraform modules: `infra/terraform/{network,compute,database,security}`
- Ansible playbooks: `infra/ansible/{provision,configure,secure}`
- CloudFormation stacks (AWS): `infra/cfn/*.yaml`

**Container Orchestration:**
- Kubernetes manifests: `k8s/{namespaces,deployments,services,ingress,hpa}`
- Docker Compose: `docker-compose.yml` with multi-stage builds
- ECS task definitions and services: `ecs/*.json`

**Application Deployments:**
- Git-based CI/CD triggers from `APP_REPO`
- Docker image builds and pushes to registry
- Helm charts: `helm/{templates,values,charts}`
- ArgoCD/Flux GitOps sync

**Database & State:**
- Liquibase/Flyway migrations: `db/migrations/{V1__*,V2__*}`
- Redis/Memcached cluster provisioning
- Backup validation with automated restore testing

**Monitoring & Health:**
- Prometheus alert rules: `monitoring/alerts.yml`
- Health check endpoints: `/health`, `/readiness`, `/live`
- Synthetic probes using curl/httpie with circuit breakers

**Excluded from scope:**
- Source code compilation (delegates to CI)
- Manual database schema edits (must be via migration files)
- Production data modifications (requires separate approval)

## Detailed Work Process

### Phase 1: Environment Discovery & Validation
```bash
# Detect current state
deployment-sorcerer discover --target=$CLOUD_PROVIDER --region=${AWS_REGION:-us-east-1}

# Validate prerequisites
deployment-sorcerer check-prereqs --strict
# Checks: kubectl context, terraform version, docker login, aws/gcp/azure auth, ansible connectivity
```

### Phase 2: Infrastructure Planning
```bash
# Generate execution plan
deployment-sorcerer plan --env=$ENVIRONMENT --tf-workspace=$ENVIRONMENT --output=plan.json

# Review drift detection
deployment-sorcerer drift-detect --compare-with=last-applied --format=json
```

### Phase 3: Infrastructure Provisioning (if needed)
```bash
# Apply infrastructure changes
deployment-sorcerer apply-infra --plan=plan.json --var="instance_type=t3.medium" --var="db_size=db.r5.large" --auto-approve

# Wait for resources
deployment-sorcerer wait --resource=eks-cluster --timeout=1800 --resource=loadbalancer --timeout=900
```

### Phase 4: Application Build & Push
```bash
# Build and tag
deployment-sorcerer build --dockerfile=deploy/Dockerfile --tag=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --build-arg=NODE_ENV=production

# Push to registry
deployment-sorcerer push --registry=$DOCKER_REGISTRY --image=$CI_REGISTRY_IMAGE --tag=$CI_COMMIT_SHA --retries=3

# Scan for vulnerabilities (optional)
deployment-sorcerer scan --image=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --severity-threshold=HIGH --fail-on-vuln=true
```

### Phase 5: Deployment Strategy Selection
Based on `DEPLOYMENT_STRATEGY` env var:
- `blue-green`: Deploy new version to separate target group, switch ELB after health checks
- `canary`: Deploy to 10% pods, gradually increase based on metrics
- `rolling`: Standard rolling update with maxUnavailable=25%
- `recreate`: Delete all then deploy (for stateful apps with maintenance window)

### Phase 6: Execute Deployment
```bash
# Blue-green example
deployment-sorcerer deploy --strategy=blue-green \
  --image=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA \
  --namespace=$APP_NAMESPACE \
  --selector="app=myapp,component=api" \
  --health-path=/health \
  --health-timeout=30 \
  --port=8080 \
  --traffic-shift-wait=300 \
  --rollback-on-failure=true

# With canary analysis
deployment-sorcerer deploy --strategy=canary \
  --analysis-interval=30 \
  --analysis-duration=600 \
  --metrics="error_rate<1%,latency_p99<500ms,throughput>100rps" \
  --prometheus-server=http://prometheus.monitoring.svc:9090
```

### Phase 7: Post-Deployment Verification
```bash
# Run smoke tests
deployment-sorcerer verify --endpoints="https://api.example.com/health,https://api.example.com/v1/status" \
  --concurrency=5 \
  --timeout=10 \
  --retries=2 \
  --expected-status=200,201

# Database connectivity check
deployment-sorcerer verify-db --query="SELECT 1" --max-latency=100ms

# External dependency validation
deployment-sorcerer verify-dependencies --service=redis,postgres,rabbitmq --timeout=30
```

### Phase 8: Monitoring & Alerting
```bash
# Deploy monitoring config
deployment-sorcerer deploy-monitoring --alerts=monitoring/alerts-$ENVIRONMENT.yml --dashboards=grafana/dashboards/

# Set up log aggregation
deployment-sorcerer setup-logs --fluentbit-config=infra/logging/fluentbit.conf --output=elasticsearch,cloudwatch
```

## Golden Rules

1. **Never deploy directly to production without a rollback plan verified** - Run `deployment-sorcerer dry-run --output=rollback-plan.json` first
2. **Always validate infrastructure drift before applying** - `deployment-sorcerer drift-detect --strict` must return empty
3. **Health checks must pass in all AZs before traffic routing** - Use `--multi-az-check=true` for multi-AZ deployments
4. **Database migrations require backup validation** - `deployment-sorcerer db-backup-validate --restore-test=true --timeout=300`
5. **Secrets must come from vault only** - Never hardcode in manifests; use `--secrets-backend=hashivault,aws-secrets-manager`
6. **Tag all resources with owner and environment** - Enforced via `infra/tags.tf` mandatory variables
7. **Zero-downtime requirement** - All deployments must maintain at least 1 healthy instance during rollout
8. **Rollback must be faster than forward deploy** - `deployment-sorcerer benchmark-rollback --max-time=300` must pass
9. **Never modify production resources outside of IaC** - All changes must go through pipeline with PR approval
10. **Post-deployment verification mandatory** - `deployment-sorcerer verify --required` must exit 0 before marking success

## Examples

### Example 1: Deploy updated API to EKS with blue-green
```bash
# User types:
export DEPLOYMENT_SORCERER_MODE=production
export CLOUD_PROVIDER=aws
export INFRASTRUCTURE_REPO=git@github.com:org/infra.git
export APP_REPO=git@github.com:org/api.git
export DOCKER_REGISTRY=123456.dkr.ecr.us-east-1.amazonaws.com
export K8S_NAMESPACE=production-api

deployment-sorcerer full-deploy \
  --app-repo=$APP_REPO \
  --branch=main \
  --commit-sha=a1b2c3d \
  --strategy=blue-green \
  --health-path=/api/v1/health \
  --health-duration=120 \
  --notify-slack="#deployments" \
  --rollback-on-failure

# Output (trimmed):
[INFO] Discovered EKS cluster: prod-cluster (us-east-1a, us-east-1b, us-east-1c)
[INFO] Building docker image: 123456.dkr.ecr.us-east-1.amazonaws.com/api:a1b2c3d
[INFO] Pushed image (layers: 15, size: 847MB)
[INFO] Creating blue deployment: api-blue-v20240115084522
[INFO] Pods rolling out: 0/12 -> 12/12 (5m12s)
[INFO] Health checks passed (100% success, avg latency: 45ms)
[INFO] Switching Route53 weighted policy: blue 0% -> green 100%
[INFO] Traffic cutover complete (300s wait)
[INFO] Running smoke tests against https://api.example.com:
  GET /health 200 ✓ (54ms)
  GET /api/v1/status 200 ✓ (78ms)
[INFO] Post-deploy verification: PASSED
[SUCCESS] Deployment complete in 11m34s. Previous version: api-green-v20240114083015 (can roll back)
```

### Example 2: Database migration with backup validation
```bash
# User types:
deployment-sorcerer db-migrate \
  --migrations-path=./db/migrations \
  --target-version=V3__add_user_table.sql \
  --backup-bucket=prod-db-backups \
  --backup-prefix=pre-migration-$(date +%Y%m%d-%H%M%S) \
  --validate-restore \
  --restore-to=temp-db-instance \
  --rollback-on-error

# Output:
[INFO] Creating backup: prod-db-backups/pre-migration-20240115-084522/backup.sql.gz (4.2GB)
[INFO] Backup SHA256: abc123... (verified)
[INFO] Testing restore to temp-db-instance... SUCCESS (2m33s)
[INFO] Running migration: V3__add_user_table.sql
[INFO] Migration status: COMPLETED (1m12s)
[INFO] Rolling back temp-db-instance cleanup...
[SUCCESS] Migration applied. Next: V4__add_indexes.sql (pending)
```

### Example 3: Drift detection and auto-remediation
```bash
# User types:
deployment-sorcerer drift-remediate \
  --resources="aws_s3_bucket.logs,aws_security_group.alb,kubernetes_deployment.api" \
  --dry-run=false \
  --generate-tf-config \
  --apply-after-reconcile

# Output:
[DRIFT] aws_s3_bucket.logs: encryption differs (expected: AES256, actual: aws:kms)
[DRIFT] kubernetes_deployment.api: replicaCount differs (expected: 6, actual: 8)
[ACTION] Generating Terraform config to reconcile...
[ACTION] Applying configuration...
[INFO] Drift remediation complete. Next scan in 10m (cron scheduled)
```

### Example 4: Multi-region deployment with traffic shifting
```bash
# User types:
deployment-sorcerer multi-region-deploy \
  --regions="us-east-1,eu-west-1,ap-southeast-1" \
  --strategy=blue-green \
  --traffic-shift-strategy=weighted \
  --initial-weight=10 \
  --step-weight=20 \
  --step-duration=600 \
  --dns-provider=route53 \
  --hosted-zone=example.com \
  --record=api.example.com

# Output:
[INFO] Deploying to us-east-1 (region 1/3)
[INFO] us-east-1: Green deployment healthy, weight 0→10%
[INFO] Deploying to eu-west-1 (region 2/3)
[INFO] eu-west-1: Green deployment healthy, weight 0→10%
[INFO] Deploying to ap-southeast-1 (region 3/3)
[INFO] ap-southeast-1: Green deployment healthy, weight 0→10%
[INFO] Global traffic distribution: us-east-1=10%, eu-west-1=10%, ap-southeast-1=10%
[INFO] Step 1/4: Increasing to 20% each region in 10m...
```

## Rollback Commands

### Immediate rollback to previous release
```bash
deployment-sorcerer rollback \
  --namespace=$APP_NAMESPACE \
  --deployment=api-server \
  --to-previous \
  --timeout=600 \
  --verify-after-rollback
```

### Rollback to specific release
```bash
deployment-sorcerer rollback \
  --deployment=api-server \
  --to-revision=deployment.api-server.v20240114083015 \
  --preserve-logs \
  --notify-slack="#deployments"
```

### Infrastructure rollback (Terraform)
```bash
deployment-sorcerer infra-rollback \
  --workspace=$ENVIRONMENT \
  --to-state=s3://tf-state/terraform.tfstate.20240114-083000 \
  --plan-only=false \
  --target-resources="aws_eks_cluster.prod,aws_rds_cluster.main"
```

### Database rollback (transactional migrations only)
```bash
deployment-sorcerer db-rollback \
  --migrations-path=./db/migrations \
  --target-version=V2__initial_schema.sql \
  --backup-restore-point=pre-migration-20240115-084522 \
  --force-if-data-loss
```

### Full application rollback (infra + app)
```bash
deployment-sorcerer full-rollback \
  --snapshot-id=snap-20240114-2300-utc \
  --infra-state=tfstate/20240114-2300 \
  --app-image=registry.example.com/api:v20240114.830 \
  --skip-db-if-no-migrations \
  --verify-rollback
```

### Partial rollback (specific microservice)
```bash
deployment-sorcerer service-rollback \
  --service=payment-processor \
  --to-version=v2.3.1 \
  --namespace=payments \
  --drain-connections=true \
  --drain-timeout=30
```

### Verify rollback success
```bash
deployment-sorcerer verify-rollback \
  --previous-deployment=api-server-v20240115084522 \
  --current-deployment=api-server-v20240114083015 \
  --endpoints=/health,/metrics \
  --error-rate-threshold=0.1% \
  --latency-threshold-p99=200ms
```

## Troubleshooting Common Issues

### Issue: Stuck deployment waiting for health checks
```bash
# Diagnose pod health
kubectl get pods -n $APP_NAMESPACE -o wide
kubectl describe pod <stuck-pod> -n $APP_NAMESPACE
kubectl logs <stuck-pod> -c container-name -n $APP_NAMESPACE

# Check service mesh sidecar (if using Istio)
kubectl logs <stuck-pod> -c istio-proxy -n $APP_NAMESPACE

# Manual health check bypass (emergency)
deployment-sorcerer override-health-fail \
  --deployment=api-server \
  --reason="Pod startup timeout - investigating" \
  --ttl=900

# Skip health checks temporarily (use with caution)
deployment-sorcerer deploy --strategy=rolling --skip-health-checks --force
```

### Issue: Insufficient resources in cluster
```bash
# Check cluster capacity
kubectl describe nodes | grep -A5 -B5 "Allocated resources"
kubectl top nodes

# Auto-scale node group (AWS EKS)
deployment-sorcerer scale-nodegroup \
  --nodegroup=prod-workers \
  --desired-size=+3 \
  --max-size=20

# Evict non-critical pods
kubectl cordon <node-with-issues>
kubectl drain <node-with-issues> --ignore-daemonsets --delete-emptydir-data
```

### Issue: Image pull failures
```bash
# Verify registry auth
deployment-sorcerer check-registry-auth --registry=$DOCKER_REGISTRY

# Refresh credentials
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $DOCKER_REGISTRY

# Use image pull secrets
kubectl create secret docker-registry regcred \
  --docker-server=$DOCKER_REGISTRY \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password --region $AWS_REGION) \
  -n $APP_NAMESPACE

# Troubleshoot specific image
deployment-sorcerer scan-image --image=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA --pull-test
```

### Issue: Terraform state drift threatening apply
```bash
# Show detailed drift
deployment-sorcerer drift-detect --detailed --json > drift-report.json

# Import drifted resources
terraform import aws_s3_bucket.logs arn:aws:s3:::prod-app-logs
terraform import kubernetes_service.api arn:aws:eks:...

# Force replace (use when import impossible)
deployment-sorcerer terraform-apply --replace-resources="aws_s3_bucket.logs,kubernetes_service.api" --with-destroy

# Emergency skip drift check (not recommended)
deployment-sorcerer apply-infra --skip-drift-check=true --force
```

### Issue: Deployment causing error rate spike
```bash
# Automatic rollback (if configured with alerts)
# This should trigger automatically via Deployment Sorcerer's monitoring

# Manual rollback if auto-rollback didn't trigger
deployment-sorcerer rollback --to-previous --verify-after-rollback

# Check metrics before redeploy
deployment-sorcerer analyze-metrics --deployment=api-server-v20240115084522 \
  --promql="sum(rate(http_requests_total{code=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))" \
  --threshold=0.05

# Deploy with canary to catch issues early
deployment-sorcerer deploy --strategy=canary --canary-percent=5 --auto-promote-if="error_rate<0.1%"
```

### Issue: Database connection pool exhaustion after deploy
```bash
# Check pod readiness and connection count
kubectl exec -n $APP_NAMESPACE <pod> -- mysqladmin -uuser -ppass processlist

# Increase connection pool in new deployment
deployment-sorcerer patch-deployment --deployment=api-server \
  --set-env=DB_POOL_MAX=50,DB_POOL_MIN=10 \
  --restart

# Emergency: increase DB max_connections
deployment-sorcerer db-execute --query="ALTER SYSTEM SET max_connections = 500;" --apply-now

# Drain old connections gradually
deployment-sorcerer restart-old-pods --deployment=api-server --batch-size=2 --interval=60
```

## Dependencies

**System packages:**
- `terraform` (>=1.5.0) - Infrastructure provisioning
- `kubectl` (>=1.28.0) - Kubernetes cluster management
- `docker` (>=24.0.0) - Container build and registry operations
- `ansible` (>=2.15.0) - Configuration management
- `jq` (>=1.6) - JSON processing
- `yq` (>=4.30.0) - YAML processing
- `curl` (>=7.68.0) - HTTP health checks
- `mysql`/`psql` client - Database connectivity verification

**Kubernetes cluster requirements:**
- Cluster with at least 3 worker nodes
- StorageClass configured for dynamic provisioning
- Ingress controller (nginx/alb/istio) installed
- HorizontalPodAutoscaler controller active
- Metrics server installed for HPA

**Cloud provider credentials:**
- AWS: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`
- GCP: `GOOGLE_APPLICATION_CREDENTIALS` pointing to service account JSON
- Azure: `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`, `AZURE_SUBSCRIPTION_ID`

**CI/CD integration:**
- Git webhook endpoint configured with `DEPLOY_WEBHOOK_SECRET`
- Docker registry access with push/pull permissions
- Slack/MS Teams webhook for notifications (optional)

## Verification Steps

```bash
# 1. Pre-deployment infrastructure health
deployment-sorcerer verify-infra \
  --check-nodes="Ready" \
  --check-pods="Running" \
  --check-storage="Bound" \
  --check-ingress="Ready"

# 2. Post-deployment smoke test suite
deployment-sorcerer run-smoke-tests \
  --suite=critical-path \
  --parallel=10 \
  --timeout=60 \
  --report-format=junit \
  --report-path=./reports/smoke-${CI_COMMIT_SHA}.xml

# 3. Full integration test (runs after smoke tests pass)
deployment-sorcerer run-integration-tests \
  --env=$ENVIRONMENT \
  --datasets=realistic \
  --cleanup-after=true \
  --timeout=600

# 4. Canary analysis (if using canary)
deployment-sorcerer canary-analyze \
  --baseline=deployment.api-server.previous \
  --candidate=deployment.api-server.current \
  --duration=900 \
  --metrics="http_error_rate,request_latency,db_connection_wait_time"

# 5. Rollback capability verification
deployment-sorcerer verify-rollback-ready \
  --deployment=api-server \
  --previous-revision=$(deployment-sorcerer get-previous-revision --deployment=api-server) \
  --rollback-timeout=300
```

## Security Considerations

- All Terraform state stored encrypted (S3 SSE-KMS or GCS CMEK)
- Docker images scanned with Trivy/Clair before deployment
- Secrets injected via external vault (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager)
- Network policies enforced: pods can only talk to whitelisted services
- Pod security standards: restricted profile (no privileged containers, read-only root FS)
- Audit logging enabled for all CLI commands (written to CloudWatch/Stackdriver)
- Role-based access control (RBAC) - deployment-sorcerer runs with scoped service account
- Immutable infrastructure - instances replaced rather than updated
- Sign and verify container images with Cosign or Notary

## Performance Optimization

- Parallel resource creation where dependencies allow
- Incremental Terraform apply with targeted resources
- Layer caching for Docker builds (`--cache-from`)
- Pre-pulled base images on cluster nodes
- Deployment batching for large microservice fleets
- Adaptive canary thresholds based on historical performance
- Resource utilization monitoring to prevent over-provisioning
- Spot instance integration with auto-healing

```