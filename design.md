# Design Document

## Goal

Build a standard kubeadm Kubernetes cluster (1 control-plane + 2 workers) and deploy a static Nginx site using a non-admin user. The user authenticates with a client certificate created through a CertificateSigningRequest, and their access is limited with RBAC to only what’s needed inside one namespace. TLS is handled by cert-manager, and the app is packaged as a Helm chart.

## Architecture

| Piece                    | Choice                              | Why                                      |
|--------------------------|-------------------------------------|------------------------------------------|
| Cluster                  | kubeadm                             | Required by the exercise                 |
| Nodes                    | 1 control-plane + 2 workers         | Matches the requirements                 |
| CNI                      | Flannel                             | Simple and reliable for this scope       |
| Ingress                  | ingress-nginx                       | Common and well-documented               |
| Certificates             | cert-manager + self-signed issuer   | Meets the TLS requirement cleanly        |
| User identity            | Client cert (`nginx-deployer`)      | Created via CSR                          |
| Authorization            | Namespace-scoped Role + RoleBinding | Least privilege                          |
| App packaging            | Helm chart                          | Satisfies the advanced / GitOps objective|

## Key Decisions

### Why kubeadm
The exercise specifically asked for a real kubeadm install instead of Minikube or kind. Using the standard tool also made the control-plane components and certificate flow more visible.

### Authentication
I created a dedicated user called `nginx-deployer` using the normal CertificateSigningRequest process:

1. Generated a private key and CSR with openssl
2. Submitted the CSR to the cluster
3. Approved it
4. Extracted the signed certificate
5. Built a kubeconfig that only uses that certificate

This stays entirely within native Kubernetes features.

### Authorization
I created a Role in the `nginx-demo` namespace that only allows the actions needed to manage the Nginx application (Deployments, Services, Pods, ConfigMaps, Secrets, and Ingresses). The RoleBinding ties that Role only to the `nginx-deployer` user and only inside that one namespace. The user has no access outside of it.

### Certificate Management
cert-manager is installed with a simple self-signed ClusterIssuer. The Ingress resource requests a certificate using the standard annotation. Once the certificate is ready, the TLS secret is automatically created and used by the Ingress.

### Application Packaging
I packaged the Nginx site as a Helm chart. This made the deployment cleaner and fulfilled the advanced objective.

## Trade-offs

**What works well**
- Everything uses native Kubernetes features
- Least-privilege is clear and easy to demonstrate
- The setup is reproducible
- Good foundation if you wanted to harden it further

**Limitations**
- Client certificates are long-lived. Rotation and revocation are manual.
- Distributing kubeconfigs with embedded certificates gets painful as the number of users grows.
- Auditability is limited compared to a real identity platform.
- No short-lived credentials, MFA, or session recording.

These are exactly the kinds of operational problems that tools like Teleport are built to solve.

## My Experience Building This

I ran everything on three Ubuntu 24.04 VMs inside Hyper-V on a Windows 11 desktop that only has 16 GB of RAM. That forced some real trade-offs.

Memory was the biggest headache. Even after setting Dynamic Memory with a 2048 MB minimum on the control plane, `free -h` was only showing about 1.9 GiB. `kubeadm init` failed with the “system RAM is less than the minimum 1700 MB” error. I ended up raising the minimum, temporarily turning Dynamic Memory off for the init, and then turning it back on later. Swap also kept coming back after every reboot until I properly commented it out in `/etc/fstab` and deleted the swap file on all three nodes.

Networking was another annoyance. DHCP kept giving the VMs new IPs, which killed my mRemoteNG sessions. I finally set static IPs (192.168.68.201 for the control plane, .202 and .203 for the workers) with Netplan so I could keep stable SSH access.

The CSR flow for the limited user felt a bit clunky the first time through, but once it was working it was satisfying to see the limited user able to create resources in `nginx-demo` while being denied access to everything else.

Overall this was a useful exercise in combining official patterns while dealing with real constraints on the host. It also made the operational downsides of long-lived client certificates very concrete.

The full set of commands used to build everything is documented in [COMMANDS.md](COMMANDS.md).

## Summary

The solution meets all the minimum requirements and includes the advanced Helm objective. It stays inside native Kubernetes features, demonstrates least privilege clearly, and surfaces the real operational limitations of long-lived client certificates
