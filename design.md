# Design Document

## 1. Goal

Build a Kubernetes cluster using kubeadm and deploy a static Nginx application using a non-administrative user. The solution demonstrates:

- Standard kubeadm installation (1 control-plane + 2 workers)
- User authentication via CertificateSigningRequest
- Least-privilege RBAC
- TLS certificate management with cert-manager
- Application packaging and deployment with Helm

## 2. Architecture Overview

| Component              | Choice                          | Notes                                      |
|------------------------|----------------------------------|--------------------------------------------|
| Cluster bootstrap      | kubeadm                         | Standard tool for production-like clusters |
| Nodes                  | 1 control-plane + 2 workers     | Matches the exercise requirements          |
| CNI                    | Flannel                         | Simple and reliable for this scope         |
| Ingress                | ingress-nginx                   | Widely used and well-documented            |
| Certificate management | cert-manager + self-signed issuer | Meets the TLS requirement cleanly        |
| Application identity   | Client certificate (`nginx-deployer`) | Created via CertificateSigningRequest |
| Authorization          | Namespace-scoped Role + RoleBinding | Least privilege within `nginx-demo`   |
| Application delivery   | Helm chart                      | Satisfies the advanced/GitOps objective    |

## 3. Key Design Decisions

### Why kubeadm?
kubeadm is the standard bootstrap tool for Kubernetes. Using it (instead of Minikube, kind, or managed services) better reflects real-world cluster creation and demonstrates understanding of the full control plane.

### Authentication
A dedicated user (`nginx-deployer`) was created using the native CertificateSigningRequest flow:

1. Generate a private key and CSR
2. Submit the CSR to the Kubernetes API
3. Approve the request
4. Extract the signed certificate
5. Build a dedicated kubeconfig

This approach relies only on built-in Kubernetes mechanisms.

### Authorization (RBAC)
A Role was created in the `nginx-demo` namespace with the minimum permissions required to manage:

- Deployments
- Services
- Pods (including logs)
- ConfigMaps
- Secrets
- Ingresses

The RoleBinding grants these permissions only to the `nginx-deployer` user and only inside the `nginx-demo` namespace. The user has no access to other namespaces or cluster-scoped resources.

### Certificate Management
cert-manager was installed and configured with a self-signed `ClusterIssuer`. The Nginx Ingress requests a certificate using the standard `cert-manager.io/cluster-issuer` annotation. This provides automatic certificate provisioning and demonstrates a realistic TLS workflow.

### Application Packaging
The Nginx application was packaged as a Helm chart. This satisfies the advanced objective and makes the deployment more repeatable and easier to manage than raw manifests alone.

## 4. Trade-offs and Limitations

**Advantages of this approach**
- Uses only native Kubernetes features
- Clear demonstration of least-privilege access
- Relatively simple to understand and reproduce
- Good foundation for further hardening

**Limitations**
- Client certificates are relatively long-lived and require manual rotation and revocation
- No short-lived or just-in-time credentials
- Limited auditability compared to a centralized identity platform
- Certificate and kubeconfig distribution becomes operationally expensive at scale
- No built-in support for MFA, session recording, or advanced policy

These limitations are common in pure certificate-based Kubernetes access models and are the type of problems that platforms like Teleport are designed to address (ephemeral certificates, centralized identity, audit logging, access requests, etc.).

## 5. Production Improvements

In a real environment the following improvements would be prioritized:

- Replace long-lived client certificates with short-lived certificates issued by a central identity system
- Enforce network policies and pod security standards
- Use a proper internal or public CA instead of a self-signed issuer
- Add a GitOps controller (Argo CD or Flux) on top of the Helm chart
- Move to a highly available control plane
- Add monitoring, alerting, and etcd backup

## 6. Summary

The solution meets all minimum requirements of the exercise and includes the advanced Helm objective. It prioritizes clarity, least privilege, and reproducibility while clearly documenting the operational limitations of certificate-based access.
