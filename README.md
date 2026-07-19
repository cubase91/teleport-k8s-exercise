# Teleport Take-Home: Kubernetes Installation & Application Deployment

This repository contains a complete solution for the Teleport Senior Field Engineer take-home exercise.

## Overview

- Kubernetes cluster built with **kubeadm** (1 control-plane + 2 worker nodes)
- Application (static Nginx site) deployed by a **non-admin user**
- User authentication via **CertificateSigningRequest** (CSR)
- Least-privilege **RBAC** scoped to a single namespace
- **cert-manager** issuing TLS certificates
- Application packaged and deployed with **Helm** (GitOps-style)

## Architecture

- **Cluster**: kubeadm on Ubuntu 24.04
- **CNI**: Flannel
- **Ingress**: ingress-nginx
- **TLS**: cert-manager with a self-signed ClusterIssuer
- **Identity**: Client certificate () + Role/RoleBinding
- **Application**: Helm chart ()

## Prerequisites

- 3 Ubuntu 24.04 nodes (or VMs) with:
  - 2 vCPU / 2–3 GB RAM each (control-plane prefers ≥ 2.5 GB)
  - Network connectivity between nodes
  - containerd + kubeadm / kubelet / kubectl installed
- Helm 3
- kubectl controls the Kubernetes cluster manager.

 Find more information at: https://kubernetes.io/docs/reference/kubectl/

Basic Commands (Beginner):
  create          Create a resource from a file or from stdin
  expose          Take a replication controller, service, deployment or pod and expose it as a new Kubernetes service
  run             Run a particular image on the cluster
  set             Set specific features on objects

Basic Commands (Intermediate):
  explain         Get documentation for a resource
  get             Display one or many resources
  edit            Edit a resource on the server
  delete          Delete resources by file names, stdin, resources and names, or by resources and label selector

Deploy Commands:
  rollout         Manage the rollout of a resource
  scale           Set a new size for a deployment, replica set, or replication controller
  autoscale       Auto-scale a deployment, replica set, stateful set, or replication controller

Cluster Management Commands:
  certificate     Modify certificate resources
  cluster-info    Display cluster information
  top             Display resource (CPU/memory) usage
  cordon          Mark node as unschedulable
  uncordon        Mark node as schedulable
  drain           Drain node in preparation for maintenance
  taint           Update the taints on one or more nodes

Troubleshooting and Debugging Commands:
  describe        Show details of a specific resource or group of resources
  logs            Print the logs for a container in a pod
  attach          Attach to a running container
  exec            Execute a command in a container
  port-forward    Forward one or more local ports to a pod
  proxy           Run a proxy to the Kubernetes API server
  cp              Copy files and directories to and from containers
  auth            Inspect authorization
  debug           Create debugging sessions for troubleshooting workloads and nodes
  events          List events

Advanced Commands:
  diff            Diff the live version against a would-be applied version
  apply           Apply a configuration to a resource by file name or stdin
  patch           Update fields of a resource
  replace         Replace a resource by file name or stdin
  wait            Experimental: Wait for a specific condition on one or many resources
  kustomize       Build a kustomization target from a directory or URL

Settings Commands:
  label           Update the labels on a resource
  annotate        Update the annotations on a resource
  completion      Output shell completion code for the specified shell (bash, zsh, fish, or powershell)

Subcommands provided by plugins:

Other Commands:
  api-resources   Print the supported API resources on the server
  api-versions    Print the supported API versions on the server, in the form of "group/version"
  config          Modify kubeconfig files
  plugin          Provides utilities for interacting with plugins
  version         Print the client and server version information

Usage:
  kubectl [flags] [options]

Use "kubectl <command> --help" for more information about a given command.
Use "kubectl options" for a list of global command-line options (applies to all commands). configured for the cluster admin

## Quick Start / Reproduction Steps

1. Prepare all nodes (swap off, kernel modules, sysctl, containerd, kubeadm packages)
2. Initialize the control plane:
   ```bash
   sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<CP-IP>
   ```
3. Install Flannel CNI
4. Join worker nodes
5. Install cert-manager and ingress-nginx
6. Create the limited user (CSR → approve → kubeconfig)
7. Apply RBAC (Role + RoleBinding) in the `nginx-demo` namespace
8. Deploy the application with Helm **as the limited user**:
   ```bash
   helm upgrade --install nginx-demo charts/nginx-demo \
     --namespace nginx-demo \
     --kubeconfig nginx-deployer.kubeconfig
   ```

Full step-by-step commands are documented in the `scripts/` directory and in `design.md`.

## Limited User & Security Model

- User: `nginx-deployer`
- Authentication: Kubernetes client certificate issued via CertificateSigningRequest
- Authorization: Role scoped only to the `nginx-demo` namespace
- Permissions: Deployments, Services, Pods, ConfigMaps, Secrets, Ingresses
- Explicitly denied: access to other namespaces and cluster-scoped resources

This demonstrates a realistic least-privilege model using native Kubernetes constructs.

## Design Decisions & Trade-offs

See [design.md](design.md) for a detailed discussion of:
- Why kubeadm was chosen
- CSR-based authentication vs other methods
- Pluses and minuses of this access model
- Choices made for CNI, Ingress, and certificate management
- How this would be improved in a real production environment (and how Teleport addresses many of the limitations)

## Project Structure

```
├── README.md
├── design.md
├── scripts/                  # Helper scripts for cluster bootstrap
├── manifests/                # Raw Kubernetes manifests (RBAC, issuer, etc.)
├── charts/nginx-demo/        # Helm chart for the application
└── kubeconfigs/              # Instructions for generating the limited-user kubeconfig
```

## Cleanup

```bash
helm uninstall nginx-demo -n nginx-demo --kubeconfig nginx-deployer.kubeconfig
kubectl delete namespace nginx-demo
kubectl delete clusterissuer selfsigned-issuer
# Then kubeadm reset on all nodes if desired
```

## Author

Prepared for the Teleport Senior Field Engineer take-home exercise.
