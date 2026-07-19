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

## Key Features

- Native Kubernetes authentication using CertificateSigningRequests
- Least-privilege access limited to a single namespace
- Automatic TLS certificate issuance via cert-manager
- Clean Helm-based application deployment
- Fully reproducible setup

## Project Structure

\`\`\`
├── README.md
├── design.md
├── charts/nginx-demo/          # Helm chart for the application
├── manifests/                  # Supporting Kubernetes manifests
│   ├── cert-manager-issuer.yaml
│   ├── namespace-rbac.yaml
│   └── limited-user/
└── scripts/
\`\`\`

## Design Decisions

Detailed discussion of architecture choices, trade-offs, and limitations is available in [design.md](design.md).

## Author

Prepared by Luke Thalmann as a technical exercise demonstrating Kubernetes installation, identity, RBAC, and application deployment practices.
