# Limited User Creation (nginx-deployer)

This directory documents how the least-privilege user was created for the Teleport take-home exercise.

## Steps Performed

1. Generated a private key and Certificate Signing Request (CSR):

   openssl genrsa -out nginx-deployer.key 2048
   openssl req -new -key nginx-deployer.key -out nginx-deployer.csr -subj "/CN=nginx-deployer"

2. Created a CertificateSigningRequest object in the cluster and approved it.

3. Extracted the signed certificate.

4. Built a dedicated kubeconfig for the `nginx-deployer` user.

5. Applied a Role and RoleBinding (see `../namespace-rbac.yaml`) that grants permissions only inside the `nginx-demo` namespace.

## Result

The `nginx-deployer` user can manage Deployments, Services, Pods, ConfigMaps, Secrets, and Ingresses inside the `nginx-demo` namespace, but has no access to other namespaces or cluster-scoped resources.
