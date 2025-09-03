# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This is a Kubernetes infrastructure repository using **Flux CD** and **Kustomize** for GitOps-based cluster management. The structure follows a layered approach:

```
clusters/
├── base/                    # Base configurations (shared across environments)
│   ├── infrastructure/      # Core infrastructure components
│   ├── postgres/           # PostgreSQL cluster templates  
│   └── source/             # Helm repositories and sources
├── prod/                   # Production environment overlays
│   ├── infrastructure/     # Production-specific patches
│   └── thux/              # Application deployments
```

## Key Technologies

- **Flux CD**: GitOps continuous delivery for Kubernetes
- **Kustomize**: Configuration management and patching
- **Helm**: Package manager via HelmRelease resources
- **CloudNativePG**: PostgreSQL operator for database clusters
- **Traefik**: Ingress controller and load balancer
- **Kyverno**: Policy engine for security and governance
- **cert-manager**: TLS certificate management
- **Kubernetes Replicator**: Secret/ConfigMap replication across namespaces

## Core Infrastructure Components

### Base Infrastructure Stack
- **DNS**: CoreDNS custom configurations + PowerDNS webhook for external DNS
- **Ingress**: Traefik with TLS termination and custom middleware
- **Security**: Kyverno policies for cluster governance  
- **Storage**: TopoLVM for local storage management
- **Scheduling**: Descheduler for workload optimization
- **Secret Management**: Kubernetes Replicator for cross-namespace secrets

### Application Architecture
- **Database**: CloudNativePG clusters with backup to S3
- **Networking**: NetworkPolicies for micro-segmentation
- **TLS**: Automated certificate management via cert-manager
- **Monitoring**: Metrics endpoints enabled on core components

## Common Commands

### Flux Operations
```bash
# Force reconcile a specific resource
flux reconcile helmrelease <name> -n <namespace>

# Check Flux system status
flux get sources all
flux get kustomizations
flux get helmreleases --all-namespaces

# Suspend/resume reconciliation
flux suspend source git <source-name>
flux resume source git <source-name>
```

### Kustomize Operations
```bash
# Preview changes before applying
kubectl kustomize clusters/prod | kubectl diff -f -

# Apply specific overlay
kubectl apply -k clusters/prod/infrastructure

# Validate kustomization structure
kubectl kustomize clusters/base --dry-run=client
```

### Database Management
```bash
# Create PostgreSQL credentials secret
kubectl create secret generic pg-credentials \
  --namespace=<target-namespace> \
  --from-literal=POSTGRES_PASSWORD="<password>" \
  --from-literal=S3_SECRET_KEY="<s3-key>"

# Check cluster status
kubectl get postgresql -n <namespace>
kubectl get pods -l postgres-operator.crunchydata.com/cluster=<cluster-name>
```

### Secret Replication
```bash
# Annotate secret for replication
kubectl annotate secret <secret-name> -n <source-namespace> \
  replicator.v1.mittwald.de/replicate-to-matching: replicator.<ref-id>=true

# Label target namespace
kubectl label namespace <target-namespace> replicator.<ref-id>=true
```

## Environment-Specific Patterns

### Production Deployment
- Uses overlay pattern in `clusters/prod/`
- Applies patches to base configurations via `kustomization.yaml`
- Includes production-specific scaling and resource limits
- Separate namespaces for workload isolation

### Configuration Layering
1. **Base layer**: Common configurations in `clusters/base/`
2. **Environment patches**: Overlays in `clusters/{env}/`
3. **Application-specific**: Further nested in environment directories

## Security Considerations

- Secrets are created manually via kubectl (not committed to git)
- Kyverno policies enforce security best practices
- NetworkPolicies provide network micro-segmentation  
- TLS encryption is mandatory for all HTTP traffic
- RBAC controls access to sensitive resources

## Troubleshooting

### Common Issues
- **Namespace not found**: Ensure namespace exists before applying resources
- **Secret not found**: Verify manual secret creation in correct namespace  
- **Chart not found**: Check HelmRepository exists in flux-system namespace
- **Kustomization failures**: Validate with `kubectl kustomize --dry-run=client`

### Debugging Commands
```bash
# Check resource status
kubectl get kustomizations.kustomize.toolkit.fluxcd.io
kubectl describe helmrelease <name> -n <namespace>

# View Flux logs
kubectl logs -n flux-system -l app=source-controller
kubectl logs -n flux-system -l app=kustomize-controller
```