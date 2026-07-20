# Kubernetes Cluster & Application Deployment Exercise

This repository contains a complete Kubernetes solution demonstrating:

- Cluster installation using **kubeadm** (1 control-plane + 2 worker nodes)
- Application deployment by a **non-administrative user**
- User authentication via **CertificateSigningRequest**
- Least-privilege **RBAC**
- TLS certificate management with **cert-manager**
- Application packaging and deployment with **Helm**

## Overview

- **Cluster**: kubeadm on Ubuntu 24.04
- **CNI**: Flannel
- **Ingress**: ingress-nginx
- **TLS**: cert-manager with a self-signed ClusterIssuer
- **Identity**: Client certificate user (`nginx-deployer`)
- **Authorization**: Namespace-scoped Role + RoleBinding
- **Application**: Helm chart (`charts/nginx-demo`)

## Documentation

- [design.md](design.md) — Architecture decisions, trade-offs, and my experience building this
- [COMMANDS.md](COMMANDS.md) — Major commands used to build the cluster and deploy the application
- [REFERENCES.md](REFERENCES.md) — Official sources and documentation used

## Prerequisites

- 3 Ubuntu 24.04 nodes (or VMs) with:
  - 2 vCPU / 2–3 GB RAM each (control-plane prefers ≥ 2.5 GB)
  - Network connectivity between nodes
  - containerd + kubeadm / kubelet / kubectl installed
- Helm 3
- `kubectl` configured for cluster-admin access

## Quick Start / Reproduction Steps

1. Prepare all nodes (disable swap, load kernel modules, configure sysctl, install containerd and Kubernetes packages)
2. Initialize the control plane:
   ```bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<CP-IP>
   ```
3. Install the Flannel CNI
4. Join the worker nodes
5. Install cert-manager and ingress-nginx
6. Create the limited user (CSR → approve → build kubeconfig)
7. Apply RBAC (Role + RoleBinding) in the `nginx-demo` namespace
8. Deploy the application with Helm **as the limited user**:
   ```bash
   helm upgrade --install nginx-demo charts/nginx-demo \
     --namespace nginx-demo \
     --kubeconfig nginx-deployer.kubeconfig
   ```

Full command details are in [COMMANDS.md](COMMANDS.md).

## Limited User & Security Model

- **User**: `nginx-deployer`
- **Authentication**: Kubernetes client certificate issued via CertificateSigningRequest
- **Authorization**: Role scoped only to the `nginx-demo` namespace
- **Permissions**: Deployments, Services, Pods, ConfigMaps, Secrets, and Ingresses
- Explicitly denied access to other namespaces and cluster-scoped resources

This demonstrates a realistic least-privilege model using only native Kubernetes constructs.

## Design Decisions & Trade-offs

See [design.md](design.md) for a detailed discussion of:

- Why kubeadm was chosen
- CSR-based authentication versus other methods
- Advantages and limitations of this access model
- Choices made for CNI, Ingress, and certificate management
- Real challenges encountered while building the environment
- How this approach would be improved in a real production environment

## Project Structure

```
├── README.md
├── design.md
├── COMMANDS.md
├── REFERENCES.md
├── charts/nginx-demo/          # Helm chart for the application
├── manifests/                  # Supporting Kubernetes manifests
│   ├── cert-manager-issuer.yaml
│   ├── namespace-rbac.yaml
│   └── limited-user/
└── scripts/
```

## Cleanup

```bash
helm uninstall nginx-demo -n nginx-demo --kubeconfig nginx-deployer.kubeconfig
kubectl delete namespace nginx-demo
kubectl delete clusterissuer selfsigned-issuer
# Optionally run kubeadm reset on all nodes
```

## Author

Prepared by Luke Thalmann as a technical exercise demonstrating Kubernetes installation, identity, RBAC, and application deployment practices.
