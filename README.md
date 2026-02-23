# AWS : Production GitOps EKS Platform using Terraform & ArgoCD
### Built by: [Bharg](https://github.com/bhargavakhil999)

[![Pre-Commit](https://github.com/bhargavakhil999/gitops-eks-blueprints/workflows/Pre-Commit/badge.svg)](https://github.com/bhargavakhil999/gitops-eks-blueprints/actions)
[![tfsec](https://github.com/bhargavakhil999/gitops-eks-blueprints/workflows/tfsec/badge.svg)](https://github.com/bhargavakhil999/gitops-eks-blueprints/actions)

# Table of Contents:
-----------------------------------------------------------
# PART A: Architecture Overview
+ Introduction
+ Solution outcomes
+ Development environment requirements
+ High-level architecture
   + EKS Blueprints architecture
   + GitOps architecture with ArgoCD
+ Karpenter — Intelligent Node Autoscaling
+ GitOps with ArgoCD
+ Logging and monitoring
+ Resources

# PART B: Deploying EKS Blueprints with Terraform
+ Main Purpose
+ Overview
+ Deployment Patterns
   + Cluster Networking
   + EKS Cluster resources
   + Karpenter Autoscaling
   + ArgoCD GitOps Add-ons
   + Logging and Monitoring
+ How to Deploy
   + Prerequisites
   + Deployment Steps
   + Example: Karpenter pattern
   + Example: GitOps pattern
+ Add-ons update
+ Important note
+ Notes

# PART C: Additional info
+ Maintainer
+ Security

-----------------------------------------------------------

# PART A: Architecture Overview

## Introduction
Kubernetes is an open-source system for automating and managing containerized applications at scale.
[Amazon Elastic Kubernetes Service (Amazon EKS)](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html) is a managed service that runs container application
workloads and helps standardize operations across environments. This project builds on top of
core EKS infrastructure by implementing a full **GitOps** workflow using [ArgoCD](https://argo-cd.readthedocs.io/en/stable/)
and intelligent node autoscaling using [Karpenter](https://karpenter.sh/).

This guide is intended for DevOps and Cloud engineers responsible for designing production-ready
EKS platforms with GitOps-driven deployments. The solution uses Terraform to provision the EKS
cluster and its add-ons, and ArgoCD to manage all subsequent application and configuration deployments
declaratively from Git.

The guide has the following key sections:
 - Development environment requirements — Software and tools required to get started.
 - High-level architecture — The overall design of the EKS GitOps platform.
 - Karpenter — How intelligent node autoscaling works and why it replaces Cluster Autoscaler.
 - GitOps with ArgoCD — How declarative deployments work and why Git is the source of truth.
 - Logging and monitoring — Observability stack used to monitor the platform.
 - Deployment patterns — Real deployable examples covering Karpenter, GitOps, multi-tenant setups.

## Solution outcomes

You should expect the following outcomes from deploying this guide's solution:
- Provision production-ready EKS clusters using reusable Terraform blueprint patterns.
- Implement GitOps with ArgoCD so that every deployment is driven by a Git commit — fully auditable and rollback-capable.
- Replace Cluster Autoscaler with Karpenter for faster, smarter, cost-optimised node provisioning.
- Deploy applications using blue/green and canary release strategies with zero downtime.
- Enable full observability with Prometheus, Grafana, and CloudWatch Container Insights.
- Implement IRSA (IAM Roles for Service Accounts) so pods access AWS services without hardcoded credentials.
- Support multi-tenant EKS clusters with namespace isolation and RBAC.

## Development environment requirements

The following table shows the tools and versions required to set up the development environment.
```
==================================================================================
Tool          Version    Purpose                    Ref()
==================================================================================
- Git         2.31.1+    Version control            (https://git-scm.com/downloads)
- Terraform   1.14.3     IaC                        (https://www.terraform.io/downloads.html)
- Helm 3      3.0.x      Kubernetes packaging       (https://helm.sh/docs/intro/install/)
- kubectl     1.34.4     Kubernetes CLI             (https://kubernetes.io/docs/tasks/tools/)
- AWS CLI     2.33.13    AWS resource management    (https://aws.amazon.com/cli/)
- ArgoCD CLI  2.x        GitOps deployments         (https://argo-cd.readthedocs.io/en/stable/cli_installation/)
```

## High-level architecture

### EKS Blueprints architecture
This project uses a **patterns-based approach** where each folder under `patterns/` is a self-contained,
deployable EKS cluster configuration. Each pattern includes its own Terraform code, variables, and
add-on configurations so you can pick exactly what you need.

```
                         ┌─────────────────────────────────────┐
                         │            AWS Cloud                 │
                         │                                      │
                         │   ┌──────────────────────────────┐   │
                         │   │        VPC (Multi-AZ)        │   │
                         │   │  ┌──────────┬─────────────┐  │   │
              Internet ──┼───┼──│  Public  │   Private   │  │   │
                ALB      │   │  │ Subnets  │   Subnets   │  │   │
                         │   │  └──────────┴─────────────┘  │   │
                         │   └──────────────────────────────┘   │
                         │                                      │
                         │   EKS Control Plane (AWS managed)    │
                         │   ┌──────────────────────────────┐   │
                         │   │       Karpenter Nodes        │   │
                         │   │  (right-sized, auto-provisioned)│  │
                         │   └──────────────────────────────┘   │
                         │                                      │
                         │   GitOps Layer (ArgoCD)              │
                         │   ┌──────────────────────────────┐   │
                         │   │  Git Push → ArgoCD detects   │   │
                         │   │  → Syncs to cluster          │   │
                         │   │  → Blue/Green or Canary      │   │
                         │   └──────────────────────────────┘   │
                         │                                      │
                         │   Observability Stack                │
                         │   Prometheus + Grafana + CloudWatch  │
                         └─────────────────────────────────────┘
```

### GitOps architecture with ArgoCD
In a traditional workflow, engineers run `kubectl apply` manually to deploy changes. In a GitOps
workflow, Git is the single source of truth. ArgoCD watches your Git repository and automatically
syncs any changes to the cluster. This means:
- Every change is a Git commit — fully auditable
- Rollback = `git revert`
- No manual kubectl commands in production
- Drift detection — if someone manually changes something in the cluster, ArgoCD detects and corrects it

## Karpenter — Intelligent Node Autoscaling

[Karpenter](https://karpenter.sh/) is an open-source, high-performance Kubernetes node provisioner
built by AWS. It replaces the traditional Cluster Autoscaler with a smarter, faster approach.

### How Karpenter works
1. A pod is created but cannot be scheduled — no nodes have enough resources
2. Karpenter sees the pending pod and evaluates its resource requirements
3. Karpenter immediately provisions the **exact right EC2 instance type** for that pod
4. The node joins the cluster and the pod runs — all within 30–60 seconds

### Karpenter vs Cluster Autoscaler

```
===================================================================================
Feature                  Cluster Autoscaler        Karpenter
===================================================================================
Provisioning speed       3–5 minutes               Under 60 seconds
Instance selection       Pre-defined node groups   Any EC2 instance type
Cost optimisation        Limited                   Automatic Spot + rightsizing
Bin packing              Basic                     Advanced (reduces waste)
Configuration            Complex ASG setup          Simple NodePool CRDs
===================================================================================
```

Karpenter's intelligent bin-packing and Spot instance support typically reduces EC2 costs by
**40–60%** compared to traditional On-Demand node groups.

## GitOps with ArgoCD

[ArgoCD](https://argo-cd.readthedocs.io/en/stable/) is a declarative GitOps continuous delivery tool
for Kubernetes. It monitors Git repositories and automatically applies changes to the cluster.

### Deployment strategies supported
- **Blue/Green** — Run two identical environments, switch traffic instantly with zero downtime
- **Canary** — Gradually shift traffic from old to new version (e.g., 10% → 50% → 100%)
- **Rolling** — Replace pods one by one with zero downtime
- **Sync waves** — Control the order in which resources are applied

### ArgoCD sync strategies
```
Sync Policy Options:
- automated.prune    = true   # Delete resources removed from Git
- automated.selfHeal = true   # Fix manual changes (drift correction)
- syncOptions:
    - CreateNamespace=true    # Auto-create namespaces
    - ApplyOutOfSyncOnly=true # Only apply changed resources
```

## Logging and monitoring

This platform includes a full observability stack:
- **Prometheus** — Collects metrics from all pods, nodes, and Kubernetes components
- **Grafana** — Visualises metrics with pre-built dashboards for EKS, Karpenter, and ArgoCD
- **CloudWatch Container Insights** — AWS-native monitoring with Fluent Bit log forwarding
- **AWS Managed Prometheus (AMP)** — Scalable, serverless Prometheus storage

## Resources
- [Karpenter Documentation](https://karpenter.sh/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/)
- [Amazon EKS Blueprints](https://aws-ia.github.io/terraform-aws-eks-blueprints/)
- [Amazon EKS Best Practices](https://aws.github.io/aws-eks-best-practices/)
- [Terraform AWS EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/)
- [GitOps with ArgoCD on EKS](https://aws.amazon.com/blogs/containers/gitops-model-for-provisioning-and-bootstrapping-amazon-eks-clusters-using-crossplane-and-argo-cd/)

-----------------------------------------------------------

# PART B: Deploying EKS Blueprints with Terraform

## Main Purpose
This project provides production-ready, reusable EKS blueprint patterns provisioned via
[Hashicorp Terraform](https://www.terraform.io/) on [AWS](https://aws.amazon.com/), with
[ArgoCD](https://argo-cd.readthedocs.io/) for GitOps deployments and
[Karpenter](https://karpenter.sh/) for intelligent node autoscaling.

## Overview
Each folder under `patterns/` is a self-contained deployable EKS pattern. Pick the pattern
that matches your use case, update the variables, and deploy. The following patterns are included:

```
patterns/
├── karpenter/                  # EKS + Karpenter node autoscaling
├── gitops/                     # EKS + ArgoCD GitOps deployments
├── multi-tenant/               # Multi-team EKS with namespace isolation
├── blue-green/                 # Zero-downtime blue/green deployments
├── fargate/                    # Serverless EKS with Fargate
└── monitoring/                 # Full Prometheus + Grafana stack
```

## Deployment Patterns

### Cluster Networking
Each pattern provisions:
1. [VPC and Subnets](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html)
   - Public Subnets (for ALB and NAT Gateway)
   - Private Subnets (for EKS nodes — never directly internet-exposed)
2. [NAT Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) — outbound internet for private nodes
3. [Internet Gateway](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
4. [VPC Endpoints](https://docs.aws.amazon.com/eks/latest/userguide/private-clusters.html) — private access to AWS services

### EKS Cluster resources
1. EKS Managed Control Plane (AWS-managed, highly available)
2. [EKS Addons](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html)
   - CoreDNS — service discovery
   - Kube-Proxy — network rules
   - VPC-CNI — pod networking
3. [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) — IAM Roles for Service Accounts (zero hardcoded credentials)
4. [RBAC](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html) — developer and admin IAM role mappings

### Karpenter Autoscaling
Karpenter is configured via `NodePool` and `EC2NodeClass` CRDs:

```hcl
# Karpenter NodePool — defines what nodes can be provisioned
resource "kubectl_manifest" "karpenter_node_pool" {
  yaml_body = <<-YAML
    apiVersion: karpenter.sh/v1beta1
    kind: NodePool
    metadata:
      name: default
    spec:
      template:
        spec:
          requirements:
            - key: karpenter.sh/capacity-type
              operator: In
              values: ["spot", "on-demand"]
            - key: kubernetes.io/arch
              operator: In
              values: ["amd64"]
      limits:
        cpu: 1000
      disruption:
        consolidationPolicy: WhenUnderutilized
  YAML
}
```

`consolidationPolicy: WhenUnderutilized` means Karpenter automatically removes underutilised
nodes and repacks workloads — directly reducing EC2 costs.

### ArgoCD GitOps Add-ons
ArgoCD is deployed via Helm through Terraform and configured to watch your Git repository:

```hcl
module "eks_blueprints_addons" {
  source = "aws-ia/eks-blueprints-addons/aws"

  enable_argocd = true
  argocd = {
    values = [templatefile("${path.module}/argocd-values.yaml", {
      github_repo = "https://github.com/bhargavakhil999/gitops-eks-blueprints"
    })]
  }
}
```

### Logging and Monitoring
```hcl
module "eks_blueprints_addons" {
  enable_aws_for_fluentbit          = true   # Logs → CloudWatch
  enable_metrics_server             = true   # Pod CPU/memory metrics
  enable_kube_prometheus_stack      = true   # Prometheus + Grafana
  enable_aws_cloudwatch_metrics     = true   # CloudWatch Container Insights
  enable_aws_load_balancer_controller = true # ALB Ingress
  enable_external_dns               = true   # Route53 DNS automation
  enable_cert_manager               = true   # SSL certificate automation
}
```

## How to Deploy

### Prerequisites
Ensure the following tools are installed:
1. [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
2. [terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
3. [kubectl](https://kubernetes.io/docs/tasks/tools/)
4. [helm](https://helm.sh/docs/intro/install/)
5. [argocd cli](https://argo-cd.readthedocs.io/en/stable/cli_installation/)

### Deployment Steps

##### Step 1: Clone the repo

```shell
git clone https://github.com/bhargavakhil999/gitops-eks-blueprints.git
cd gitops-eks-blueprints
```

##### Step 2: Configure AWS credentials

```shell
aws configure
# Enter: Access Key ID, Secret Access Key, Region (us-east-1), Output (json)
```

##### Step 3: Choose a pattern and initialise Terraform

```shell
cd patterns/karpenter   # or gitops, multi-tenant, etc.

terraform init
```

##### Step 4: Preview the changes

```shell
terraform plan
```

##### Step 5: Deploy

```shell
terraform apply
```

Type `yes` when prompted. Deployment takes approximately 15–20 minutes.

##### Step 6: Connect kubectl to your cluster

```shell
aws eks --region us-east-1 update-kubeconfig --name <cluster-name>
kubectl get nodes
kubectl get pods -A
```

##### Step 7: Access ArgoCD UI

```shell
# Get the ArgoCD URL
kubectl get svc -n argocd argocd-server

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Login via CLI
argocd login <ARGOCD_URL> --username admin --password <PASSWORD>
```

##### Step 8: Verify Karpenter is running

```shell
kubectl get pods -n karpenter
kubectl get nodepools
kubectl get ec2nodeclasses
```

##### Step 9: Verify autoscaling works

```shell
# Deploy a test workload to trigger Karpenter
kubectl create deployment inflate --image=public.ecr.aws/eks-distro/kubernetes/pause:3.7 --replicas=5
kubectl get nodes --watch   # Watch Karpenter provision a new node
```

##### Step 10: Clean up

```shell
terraform destroy
```

### Example: Karpenter pattern

```shell
$ cd patterns/karpenter
$ terraform init && terraform apply

...
module.karpenter.aws_iam_role.karpenter: Creation complete
module.eks.aws_eks_cluster.this: Creation complete after 12m
module.karpenter.helm_release.karpenter: Creation complete after 45s

Apply complete! Resources: 78 added, 0 changed, 0 destroyed.

$ aws eks update-kubeconfig --region us-east-1 --name bharg-karpenter-cluster
$ kubectl get nodes
NAME                          STATUS   ROLES    AGE   VERSION
ip-10-0-1-45.compute.internal Ready    <none>   2m    v1.29.0-eks-abc123

$ kubectl get pods -n karpenter
NAME                        READY   STATUS    RESTARTS   AGE
karpenter-7d9f8b6c5-xk2pl   1/1     Running   0          3m
```

### Example: GitOps pattern with ArgoCD

```shell
$ cd patterns/gitops
$ terraform init && terraform apply

...
module.eks_blueprints_addons.helm_release.argocd: Creation complete after 1m30s

Apply complete! Resources: 92 added, 0 changed, 0 destroyed.

$ kubectl get pods -n argocd
NAME                                       READY   STATUS    RESTARTS   AGE
argocd-application-controller-0            1/1     Running   0          2m
argocd-repo-server-6d7b9f8c4-xp2ml         1/1     Running   0          2m
argocd-server-5f8b9d7c6-kl3mn              1/1     Running   0          2m

$ kubectl get applications -n argocd
NAME              SYNC STATUS   HEALTH STATUS
cluster-addons    Synced        Healthy
workloads         Synced        Healthy
```

## Add-ons update
AWS EKS doesn't modify add-ons when you update a cluster version. Always upgrade add-ons
alongside cluster upgrades. Update add-on versions in the respective Terraform variables and
run `terraform apply` to upgrade.

See [AWS documentation](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html) for full cluster upgrade instructions.

## Important note
Always check [Karpenter compatibility](https://karpenter.sh/docs/upgrading/compatibility/) and
[ArgoCD release notes](https://github.com/argoproj/argo-cd/releases) before upgrading versions.
Ensure your application Helm charts are compatible with the target Kubernetes version before
upgrading the cluster.

## Notes
Required VPC tags for EKS load balancer discovery:

```hcl
public_subnet_tags = {
  "kubernetes.io/cluster/${local.cluster_name}" = "shared"
  "kubernetes.io/role/elb"                      = "1"
}

private_subnet_tags = {
  "kubernetes.io/cluster/${local.cluster_name}" = "shared"
  "kubernetes.io/role/internal-elb"             = "1"
}
```

Required VPC endpoints for private EKS clusters:
```
com.amazonaws.region.ec2
com.amazonaws.region.ecr.api
com.amazonaws.region.ecr.dkr
com.amazonaws.region.s3
com.amazonaws.region.sts
com.amazonaws.region.logs
com.amazonaws.region.autoscaling
com.amazonaws.region.elasticloadbalancing
```

-----------------------------------------------------------

# PART C: Additional info

## Maintainer
Maintained by [Bharg](https://github.com/bhargavakhil999)

## Security
See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License
This library is licensed under the Apache-2.0 License. See the LICENSE file.