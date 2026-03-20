# eks-production

> Production Kubernetes cluster on AWS EKS — managed control plane, multi-AZ node groups, ALB Ingress, Cluster Autoscaler, IRSA, and Terraform-provisioned infrastructure.

Part of a two-tier Kubernetes strategy:

| Repo | Environment | Platform | Status |
|---|---|---|---|
| [k3s-dev-staging](https://github.com/Vishal-B142/k3s-dev-staging) | Dev & Staging | k3s on Hostinger VPS | ✅ Live |
| **eks-production** (this repo) | Production | AWS EKS | 🔄 In Progress |

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                        AWS EKS                           │
│                                                          │
│   Route53 ──▶ ALB ──▶ ALB Ingress Controller            │
│                              │                           │
│   ┌───────────────────────────▼──────────────────────┐   │
│   │         EKS Managed Node Groups                  │   │
│   │   ┌─────────────┐   ┌─────────────┐              │   │
│   │   │ ap-south-1a │   │ ap-south-1b │  Multi-AZ    │   │
│   │   └─────────────┘   └─────────────┘              │   │
│   └──────────────────────────────────────────────────┘   │
│                          │                               │
│   ┌──────────────────────▼────────────────────────────┐  │
│   │  Namespaces                                       │  │
│   │  ┌────────────────┐   ┌──────────────────────┐   │  │
│   │  │   production   │   │     monitoring       │   │  │
│   │  └────────────────┘   └──────────────────────┘   │  │
│   └───────────────────────────────────────────────────┘  │
│                                                          │
│   Cluster Autoscaler │ HPA │ IRSA │ CloudWatch Logs      │
└──────────────────────────────────────────────────────────┘
```

---

## Stack

![EKS](https://img.shields.io/badge/AWS_EKS-232F3E?style=flat&logo=amazonaws&logoColor=white)
![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat&logo=terraform&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=flat&logo=amazonaws&logoColor=white)

---

## Repo Structure

```
eks-production/
├── terraform/
│   ├── main.tf                  # EKS cluster + VPC + node groups
│   ├── variables.tf
│   ├── outputs.tf
│   ├── backend.tf               # Remote state — S3 + DynamoDB locking
│   └── modules/
│       ├── eks/
│       ├── vpc/
│       └── iam/
├── namespaces/
│   ├── production.yaml
│   └── monitoring.yaml
├── alb-ingress/
│   ├── values.yaml              # AWS Load Balancer Controller Helm values
│   └── ingress-example.yaml
├── autoscaler/
│   └── cluster-autoscaler.yaml
├── irsa/
│   └── service-account.yaml     # IAM Roles for Service Accounts
├── resource-quotas/
│   └── production-quota.yaml
├── hpa/
│   └── hpa-example.yaml        # Horizontal Pod Autoscaler
└── README.md
```

---

## Terraform — EKS Cluster

```hcl
module "eks" {
  source          = "terraform-aws-modules/eks/aws"
  cluster_name    = "prod-cluster"
  cluster_version = "1.29"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    production = {
      min_size       = 2
      max_size       = 6
      desired_size   = 2
      instance_types = ["t3.medium"]
    }
  }
}
```

## Setup

```bash
# 1. Provision EKS via Terraform
cd terraform
terraform init
terraform plan
terraform apply

# 2. Configure kubectl
aws eks update-kubeconfig --region ap-south-1 --name prod-cluster

# 3. Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system --set clusterName=prod-cluster

# 4. Install Cluster Autoscaler
kubectl apply -f autoscaler/cluster-autoscaler.yaml

# 5. Apply namespaces and quotas
kubectl apply -f namespaces/
kubectl apply -f resource-quotas/
```

---

## Planned Features

- [x] EKS cluster Terraform module
- [x] Multi-AZ node groups
- [x] ALB Ingress Controller
- [ ] Cluster Autoscaler
- [ ] HPA per service
- [ ] IRSA — zero static credentials
- [ ] CloudWatch Container Insights
- [ ] Migrate workloads from k3s staging

---

## Related

- [k3s-dev-staging](https://github.com/Vishal-B142/k3s-dev-staging) — dev and staging cluster
- [jenkins-k8s-pipeline](https://github.com/Vishal-B142/jenkins-k8s-pipeline) — CI/CD deploying to this cluster
- [terraform-aws-infra](https://github.com/Vishal-B142/terraform-aws-infra) — shared AWS Terraform modules
- [observability-stack](https://github.com/Vishal-B142/observability-stack) — monitoring for production
