# eks-production

> Production Kubernetes cluster on AWS EKS — migration from k3s VPS. This runbook will be updated once the production cluster is live.

Part of a two-tier Kubernetes strategy:

| Repo | Environment | Platform | Status |
|---|---|---|---|
| [k3s-dev-staging](https://github.com/Vishal-B142/k3s-dev-staging) | Dev & Staging | k3s on VPS | ✅ Live |
| **eks-production** (this repo) | Production | AWS EKS | 🔄 In Progress |

---

## Planned Architecture

```
                          Internet
                              │
                              ▼
                    Route 53 (DNS)
                              │
                              ▼
                   AWS ALB (ALB Ingress Controller)
                    SSL termination via ACM
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
   api.example.com     ai.example.com    grafana.example.com  ...
          │
          ▼
┌─────────────────────────────────────────────────────┐
│  EKS Managed Node Groups  (ap-south-1)              │
│                                                     │
│  ┌─────────────────────┐  ┌─────────────────────┐  │
│  │   prod-apps         │  │   prod-monitoring   │  │
│  │   t3.large  min=2   │  │   t3.medium  min=1  │  │
│  │   max=6  role=apps  │  │   max=2  role=mon   │  │
│  └─────────────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│  production namespace                               │
│  Blue / Green deployments (same as k3s)             │
│  HPA — CPU + Memory thresholds                      │
└─────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│  monitoring namespace                               │
│  Prometheus + Grafana (EBS gp2 persistence)         │
│  Loki (S3 backend)  +  Fluent Bit DaemonSet         │
└─────────────────────────────────────────────────────┘
          │
          ▼
┌────────────────────────┐   ┌────────────────────────┐
│   AWS ECR              │   │   AWS Secrets Manager  │
│   Container registry   │   │   + External Secrets   │
│   (already in use)     │   │   Operator (Phase 2)   │
└────────────────────────┘   └────────────────────────┘
```

---

## Key Differences from k3s Dev/Staging

| Component | k3s (dev/staging) | EKS (production) |
|---|---|---|
| Ingress | Traefik + MetalLB | AWS ALB Ingress Controller |
| SSL | certbot + Let's Encrypt | AWS ACM (auto-renews) |
| Storage | emptyDir / local-path | EBS gp2 (Grafana, Prometheus) |
| Logs backend | Filesystem | S3 |
| Log shipper | Promtail | Fluent Bit |
| Secrets | k8s Secrets (base64) | AWS Secrets Manager + ESO |
| HA | Single VPS node | Multi-AZ node groups |
| Autoscaling | HPA only | HPA + Cluster Autoscaler |

---

## Migration Checklist

- [ ] Create EKS cluster with eksctl
- [ ] Install AWS Load Balancer Controller
- [ ] Request ACM wildcard certificate
- [ ] Update ingress annotations (Traefik → ALB)
- [ ] Apply deployments + HPA (unchanged from k3s)
- [ ] Update monitoring-values.yaml for EBS persistence
- [ ] Update loki-values.yaml for S3 backend
- [ ] Smoke test via port-forward
- [ ] Lower Route 53 TTLs to 60s (24hrs before cutover)
- [ ] Switch Route 53 → ALB DNS
- [ ] Monitor 72 hours → decommission VPS

---

## 🚧 This README will be updated with full runbook, YAML configs, and architecture diagram once the production cluster is live.

---

## Related

- [k3s-dev-staging](https://github.com/Vishal-B142/k3s-dev-staging) — current live cluster (dev & staging)
- [jenkins-k8s-pipeline](https://github.com/Vishal-B142/jenkins-k8s-pipeline) — CI/CD pipeline
- [terraform-aws-infra](https://github.com/Vishal-B142/terraform-aws-infra) — Terraform modules
- [observability-stack](https://github.com/Vishal-B142/observability-stack) — monitoring stack
