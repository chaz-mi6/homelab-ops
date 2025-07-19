# Kubernetes Infrastructure as Code

This directory contains the complete GitOps infrastructure for the homelab k3s cluster, organized for deployment via ArgoCD.

## Architecture Overview

### Core Components

- **ArgoCD**: GitOps operator with high availability, SSO integration via Vault OIDC
- **External Secrets Operator**: Manages secrets from HashiCorp Vault
- **cert-manager**: Automated SSL certificate management with Let's Encrypt and Cloudflare DNS
- **Traefik**: Ingress controller with automatic SSL termination and load balancing

### Monitoring & Observability

- **Prometheus Stack**: Metrics collection with custom alerting rules and long-term storage
- **Thanos**: Long-term Prometheus metrics storage and global querying
- **Grafana**: Visualization dashboards with OIDC authentication and automated provisioning
- **Loki + Promtail**: Centralized logging with retention policies and log parsing
- **AlertManager**: Alert routing with webhook integrations and escalation policies
- **Linkerd**: Service mesh for traffic management, security, and observability

### Backup & Recovery

- **Velero**: Kubernetes cluster backup with CSI snapshot support
- **Minio**: S3-compatible storage for backup retention
- **ETCD Backup**: Automated control plane backup with retention

## Directory Structure

```
infrastructure/k8s/
├── apps/              # ArgoCD App of Apps pattern
├── backup/           # Backup and recovery systems
├── core/             # Core Kubernetes resources and namespaces
├── ingress/          # Ingress controllers and service mesh
├── logging/          # Centralized logging infrastructure
├── monitoring/       # Metrics, alerting, and observability
├── security/         # Security tooling and certificate management
└── storage/          # Storage classes and volume management
```

## Deployment Order

The infrastructure is designed to deploy in the following order via ArgoCD:

1. **Core Infrastructure** (`core/`)
   - Namespaces
   - ArgoCD installation and configuration
   - RBAC and service accounts

2. **Security Infrastructure** (`security/`)
   - External Secrets Operator
   - cert-manager with Let's Encrypt issuers
   - Vault integration and secret stores

3. **Ingress Infrastructure** (`ingress/`)
   - Traefik ingress controller
   - Linkerd service mesh (optional)
   - Network policies and traffic routing

4. **Monitoring Infrastructure** (`monitoring/`)
   - Prometheus operator and stack
   - Thanos for long-term storage
   - Grafana with dashboard provisioning
   - AlertManager with webhook integration

5. **Logging Infrastructure** (`logging/`)
   - Loki for log aggregation
   - Promtail for log collection
   - Log parsing and retention policies

6. **Backup Infrastructure** (`backup/`)
   - Minio for object storage
   - Velero for cluster backups
   - ETCD backup automation

## Getting Started

### Prerequisites

1. **k3s cluster** running and accessible
2. **HashiCorp Vault** configured with:
   - Kubernetes auth method
   - OIDC provider for SSO
   - Required secrets stored at expected paths
3. **Cloudflare API token** for DNS validation
4. **Git repository** with this infrastructure code

### Initial Deployment

1. **Install ArgoCD manually** (bootstrap):
   ```bash
   kubectl apply -n argocd -f core/namespace.yaml
   kubectl apply -n argocd -f core/argocd-install.yaml
   kubectl apply -n argocd -f core/argocd-rbac.yaml
   ```

2. **Configure ArgoCD to track this repository**:
   ```bash
   kubectl apply -n argocd -f core/argocd-bootstrap.yaml
   ```

3. **Deploy the App of Apps**:
   ```bash
   kubectl apply -n argocd -f apps/app-of-apps.yaml
   ```

### Accessing Services

After successful deployment, services will be available at:

- **ArgoCD**: `https://argocd.homelab.local`
- **Grafana**: `https://grafana.homelab.local`
- **Prometheus**: `https://prometheus.homelab.local`
- **AlertManager**: `https://alertmanager.homelab.local`
- **Traefik Dashboard**: `https://traefik.homelab.local`
- **Linkerd Dashboard**: `https://linkerd.homelab.local`
- **Minio Console**: `https://minio.homelab.local`
- **Loki**: `https://loki.homelab.local`

### Configuration Requirements

#### Vault Secrets

The following secrets must be configured in Vault:

```
# ArgoCD OIDC
secret/argocd:
  oidc_client_secret: <vault-oidc-client-secret>

# Cloudflare DNS
secret/cloudflare:
  api_token: <cloudflare-api-token>

# Grafana
secret/grafana:
  admin_password: <admin-password>
  oidc_client_secret: <vault-oidc-client-secret>

# Minio
secret/minio:
  root_password: <root-password>
  velero_password: <velero-user-password>
  velero_access_key_id: <velero-access-key>
  velero_secret_access_key: <velero-secret-key>

# Backup webhooks
secret/backup:
  webhook_url: <notification-webhook-url>
  webhook_token: <webhook-auth-token>
```

#### Vault Auth Method

Configure Kubernetes auth method in Vault:

```bash
# Enable Kubernetes auth
vault auth enable kubernetes

# Configure the auth method
vault write auth/kubernetes/config \
    kubernetes_host="https://<k8s-api-server>:6443" \
    kubernetes_ca_cert=@ca.crt

# Create policy and role for external-secrets
vault policy write external-secrets-policy - <<EOF
path "secret/data/*" {
  capabilities = ["read"]
}
EOF

vault write auth/kubernetes/role/external-secrets \
    bound_service_account_names=external-secrets-sa \
    bound_service_account_namespaces=external-secrets \
    policies=external-secrets-policy \
    ttl=24h
```

## High Availability Features

- **ArgoCD**: Multi-replica deployment with Redis HA
- **Prometheus**: 2 replicas with external label configuration
- **AlertManager**: 2 replicas with clustering
- **Grafana**: Single replica with persistent storage
- **Traefik**: 2+ replicas with auto-scaling
- **Linkerd**: Control plane HA with 2 replicas

## Security Features

- **mTLS everywhere** via Linkerd service mesh
- **RBAC** configured for all services
- **Network policies** restricting inter-namespace communication
- **Secret management** via External Secrets Operator and Vault
- **SSL/TLS** automatic certificate management
- **OIDC authentication** for all web interfaces
- **Security scanning** and policy enforcement

## Monitoring & Alerting

- **Infrastructure metrics** via node-exporter and kube-state-metrics
- **Application metrics** via service monitors
- **Custom alerting rules** for infrastructure health
- **Multi-channel notifications** (email, Slack, webhooks)
- **Grafana dashboards** for visualization
- **Long-term metrics storage** via Thanos

## Backup Strategy

- **Daily incremental backups** of persistent volumes
- **Weekly full cluster backups** including all namespaces
- **ETCD snapshots** with automated uploading to object storage
- **30-day retention** for backups
- **Backup monitoring** with alerts on failures

## Troubleshooting

### Common Issues

1. **ArgoCD sync failures**: Check RBAC permissions and secret availability
2. **Certificate issues**: Verify Cloudflare API token and DNS propagation
3. **Monitoring gaps**: Check service monitor labels and Prometheus configuration
4. **Backup failures**: Verify Minio connectivity and storage capacity
5. **Linkerd issues**: Check proxy injection and certificate rotation

### Useful Commands

```bash
# Check ArgoCD application status
kubectl get applications -n argocd

# View ArgoCD sync status
argocd app list

# Check certificate status
kubectl get certificates --all-namespaces

# Monitor backup status
kubectl get backups -n backup

# Linkerd service mesh check
linkerd check
```

## Maintenance

### Regular Tasks

- **Certificate renewal**: Automated via cert-manager
- **Backup verification**: Weekly restore tests
- **Security updates**: Monthly image updates
- **Capacity planning**: Monitor resource usage and storage
- **Log retention**: Automated cleanup via retention policies

### Scaling

- **Horizontal scaling**: Adjust replica counts in values files
- **Vertical scaling**: Update resource requests and limits
- **Storage scaling**: Expand PVC sizes as needed
- **Node scaling**: Add worker nodes for increased capacity

This infrastructure provides a production-ready, highly available, and secure Kubernetes platform with comprehensive monitoring, logging, and backup capabilities.