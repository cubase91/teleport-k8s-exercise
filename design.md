# Design Document – Teleport Take-Home Exercise

## 1. Goal

Build a Kubernetes cluster using kubeadm and deploy a static Nginx application using a non-administrative user. The solution must demonstrate:

- Standard kubeadm installation (1 control-plane + 2 workers)
- User authentication via CertificateSigningRequest
- Least-privilege RBAC
- TLS certificate management with cert-manager
- Clean, reproducible deployment (including Helm as the advanced/GitOps option)

## 2. High-Level Architecture

- **Infrastructure**: Three Ubuntu 24.04 nodes (Hyper-V)
- **Cluster Bootstrap**: kubeadm
- **CNI**: Flannel
- **Ingress Controller**: ingress-nginx
- **Certificate Management**: cert-manager with a self-signed ClusterIssuer
- **Application Identity**: Client certificate user (`nginx-deployer`)
- **Authorization**: Namespace-scoped Role + RoleBinding
- **Application Delivery**: Helm chart (`charts/nginx-demo`)

## 3. Key Design Decisions

### Why kubeadm?
kubeadm is the standard tool for bootstrapping production-like Kubernetes clusters. Using it (instead of Minikube, kind, or k3s) better matches real-world environments and demonstrates understanding of the full control-plane components.

### Authentication Model
A dedicated user (`nginx-deployer`) was created using the native CertificateSigningRequest flow:

1. Generate private key + CSR
2. Submit CSR to the Kubernetes API
3. Approve the request
4. Extract the signed certificate
5. Build a dedicated kubeconfig

This approach uses only built-in Kubernetes mechanisms and avoids external identity providers for the scope of this exercise.

### Authorization (RBAC)
A Role was created in the `nginx-demo` namespace with the minimum permissions required to:
- Manage Deployments, Services, Pods, ConfigMaps, Secrets, and Ingresses

The corresponding RoleBinding binds this Role only to the `nginx-deployer` user.  
The user has no permissions outside the `nginx-demo` namespace.

### Certificate Management
cert-manager was installed and configured with a self-signed `ClusterIssuer`.  
The Nginx Ingress resource requests a certificate via the standard `cert-manager.io/cluster-issuer` annotation. This provides automatic certificate creation and demonstrates a realistic TLS workflow.

### Application Packaging (Advanced Objective)
The Nginx application was packaged as a Helm chart. This satisfies the GitOps-style deployment requirement and makes the application easier to version, parameterize, and promote across environments.

## 4. Trade-offs and Limitations of This Approach

**Advantages**
- Uses only native Kubernetes constructs
- Clear demonstration of least-privilege access
- Reproducible and easy to reason about
- Good foundation for further hardening

**Limitations**
- Client certificates are relatively long-lived and require manual rotation/revocation
- No short-lived credentials or just-in-time access
- Limited auditability compared to a centralized identity platform
- Certificate and kubeconfig distribution becomes operationally heavy at scale
- No built-in session recording, MFA, or fine-grained policy beyond RBAC

These limitations are exactly the class of problems that Teleport is designed to solve (ephemeral certificates, SSO integration, audit logging, access requests, etc.).

## 5. What Would Be Improved in Production

- Replace long-lived client certificates with short-lived certificates issued through a central identity platform
- Add network policies and Pod Security admission
- Use a proper internal or public CA instead of a self-signed issuer
- Introduce GitOps controllers (Argo CD or Flux) on top of the Helm chart
- Add monitoring, alerting, and backup for etcd
- Move to a highly available control plane

## 6. Conclusion

The solution meets all minimum requirements of the take-home exercise and includes the advanced Helm/GitOps objective. It prioritizes clarity, least privilege, and reproducibility while honestly acknowledging the operational limitations of pure certificate-based access — limitations that motivate platforms like Teleport.
