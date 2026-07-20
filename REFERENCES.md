# References

These are the main sources I used while putting this together. I pulled from official docs for the core patterns (kubeadm, CSRs, RBAC, cert-manager, etc.) and then adapted everything to fit this exercise.

## Kubernetes Official Docs
- kubeadm cluster creation:  
  https://github.com/kubernetes/website/blob/main/content/en/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm.md

- Certificate management with kubeadm:  
  https://github.com/kubernetes/website/blob/main/content/en/docs/tasks/administer-cluster/kubeadm/kubeadm-certs.md

- CertificateSigningRequests:  
  https://github.com/kubernetes/website/blob/main/content/en/docs/reference/access-authn-authz/certificate-signing-requests.md

- RBAC:  
  https://github.com/kubernetes/website/blob/main/content/en/docs/reference/access-authn-authz/rbac.md

- Certificate best practices:  
  https://github.com/kubernetes/website/blob/main/content/en/docs/setup/best-practices/certificates.md

## cert-manager
- Main repo and docs:  
  https://github.com/cert-manager/cert-manager

## ingress-nginx
- Official chart and configuration:  
  https://github.com/kubernetes/ingress-nginx

I also looked at a handful of community examples and labs while troubleshooting, but the core patterns came from the official sources above. Everything in this repo was customized for this specific exercise (namespace design, the limited user, the Helm chart, etc.).
