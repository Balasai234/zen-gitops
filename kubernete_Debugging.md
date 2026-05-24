README.md — GitOps Troubleshooting & Kubernetes Debugging
# GitOps Troubleshooting & Kubernetes Debugging Notes

## Project Overview

This project demonstrates a complete GitOps-based CI/CD deployment pipeline using:

- GitHub Actions
- Docker
- AWS ECR
- Amazon EKS
- ArgoCD
- Helm
- External Secrets Operator
- AWS Secrets Manager
- Spring Boot Microservices
- PostgreSQL RDS

---

# Architecture Flow

Developer Push
↓
GitHub Actions CI Pipeline
↓
Build Docker Image
↓
Push Image to AWS ECR
↓
Update GitOps Repository
↓
ArgoCD Detects Git Change
↓
Helm Deploys to EKS
↓
Kubernetes Runs Application

---

# Issues Faced & Resolutions

---

# 1. Git Push Authentication Failure

## Issue

Git push failed from EC2 instance.

Error:

```bash
remote: Invalid username or token.
fatal: Authentication failed
Root Cause

GitHub password authentication is deprecated.

Resolution

Used GitHub Personal Access Token (PAT).

git push https://<PAT>@github.com/Balasai234/zen-pharma-backend.git develop
2. GitHub Actions Secret Missing
Issue

Pipeline failed because AWS secrets were missing.

Resolution

Added GitHub Secrets:

AWS_ACCOUNT_ID
AWS_REGION
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
3. GitOps Repository Checkout Failure
Issue

GitHub Actions failed during checkout step.

Error:

Invalid repository format
Root Cause

Incorrect repository format used.

Wrong
repository: https://github.com/Balasai234/zen-gitops
Correct
repository: Balasai234/zen-gitops
4. ArgoCD RepoURL Misconfiguration
Issue

ArgoCD application showed:

Synced
Degraded

Pods were not updating.

Root Cause

ArgoCD Application pointed to old GitOps repository.

Wrong
repoURL: https://github.com/DPP-2026/zen-gitops.git
Correct
repoURL: https://github.com/Balasai234/zen-gitops.git
Resolution

Updated ArgoCD Application YAML and re-applied.

kubectl apply -f auth-service-app.yaml
5. AppProject Permission Issue
Issue

ArgoCD Application failed with:

application repo is not permitted in project
Root Cause

Repository not whitelisted in AppProject.

Resolution

Updated AppProject:

sourceRepos:
  - "https://github.com/Balasai234/zen-gitops.git"

Applied:

kubectl apply -f pharma-project.yaml
6. ImagePullBackOff Error
Issue

Pods failed with:

ImagePullBackOff
ErrImagePull
Root Cause

Incorrect AWS account ID in ECR repository.

Wrong
repository: 516209541629.dkr.ecr.us-east-1.amazonaws.com/auth-service
Correct
repository: 243746944939.dkr.ecr.us-east-1.amazonaws.com/auth-service
Resolution

Updated Helm values file in GitOps repository.

7. ECR IAM Permission Verification
Investigation

Verified EKS node role permissions.

aws iam list-attached-role-policies \
--role-name pharma-dev-eks-node-group-role

Verified policy existed:

AmazonEC2ContainerRegistryReadOnly
8. External Secrets Failure
Issue

Secrets were not created in Kubernetes.

Error:

SecretSyncedError
Root Cause

ClusterSecretStore resource missing.

Resolution

Created ClusterSecretStore.

kind: ClusterSecretStore

Verified:

kubectl get clustersecretstore
9. Secret Not Found Error
Issue

Pod failed with:

CreateContainerConfigError

Error:

secret "db-credentials" not found
Root Cause

External Secrets Operator could not sync secrets.

Resolution

Applied ExternalSecret resources:

kubectl apply -f db-secret.yaml -n dev
kubectl apply -f jwt-secret.yaml -n dev

Verified:

kubectl get externalsecrets -n dev
kubectl get secrets -n dev
10. RDS Endpoint DNS Failure
Issue

Application continuously restarted.

Logs showed:

java.net.UnknownHostException
Root Cause

Old RDS endpoint configured inside ConfigMap.

Wrong
pharma-dev-postgres.cyrywaguk6v4.us-east-1.rds.amazonaws.com
Correct
pharma-dev-postgres.cupgoqko4dpc.us-east-1.rds.amazonaws.com
Resolution

Updated GitOps values file with correct RDS endpoint.

11. Why Pods Did Not Restart Automatically
Important Kubernetes Behavior

ArgoCD successfully synced ConfigMap changes.

However, Kubernetes does NOT automatically restart pods when:

ConfigMaps change
Secrets change

Pods load environment variables only during startup.

Resolution

Performed rollout restart:

kubectl rollout restart deployment auth-service -n dev

After restart:

New pod loaded updated ConfigMap
Application connected successfully to RDS
Final Working Status
Auth Service
READY   STATUS
1/1     Running
Important Commands Used
Check ArgoCD Applications
kubectl get applications -n argocd
Describe Pod
kubectl describe pod <pod-name> -n dev
Check Logs
kubectl logs <pod-name> -n dev
Check Deployment Image
kubectl get deployment auth-service -n dev -o yaml | grep image
Restart Deployment
kubectl rollout restart deployment auth-service -n dev
Verify Secrets
kubectl get secrets -n dev
Verify External Secrets
kubectl get externalsecrets -n dev
Key Learnings
ArgoCD Responsibility

ArgoCD:

Syncs manifests from Git
Reconciles cluster state

ArgoCD does NOT:

Fix infrastructure issues
Restart pods for ConfigMap changes
Fix IAM issues
Fix application runtime failures
Kubernetes Behavior
Change	Automatic Restart
Image Change	YES
Deployment Spec Change	YES
ConfigMap Update	NO
Secret Update	NO
Production Troubleshooting Flow

Typical debugging sequence:

ImagePullBackOff
↓
Fix image repository
↓
CreateContainerConfigError
↓
Fix secrets
↓
CrashLoopBackOff
↓
Fix application configuration
↓
Database connectivity fix
↓
Application healthy
Final Outcome

Successfully debugged and resolved:

GitHub authentication issues
GitHub Actions failures
ArgoCD sync issues
AppProject permission problems
ECR image pull failures
IAM permission validation
External Secrets synchronization failures
Missing Kubernetes secrets
ConfigMap drift
RDS DNS resolution issue
Pod restart behavior understanding

This troubleshooting exercise simulated a real-world production GitOps debugging scenario.