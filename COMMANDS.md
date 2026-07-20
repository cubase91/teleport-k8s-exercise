# Key Commands Used

This file contains the major commands used to build the cluster and deploy the application.

## 1. Node Preparation (run on all three nodes)

```bash
# Disable swap permanently
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Load required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# Configure sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

# Install and configure containerd
sudo apt update
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

## 2. Install Kubernetes Packages (run on all three nodes)

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable kubelet
```

## 3. Initialize the Control Plane

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=192.168.68.201
```

After a successful init, run:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Install the Flannel CNI:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

## 4. Join the Worker Nodes

On each worker node, run the `kubeadm join` command that was printed at the end of `kubeadm init` on the control plane.

## 5. Install cert-manager and ingress-nginx

```bash
# Add the Jetstack repo and install cert-manager
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.17.2 \
  --set crds.enabled=true
```

```bash
# Install ingress-nginx
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

Create the self-signed ClusterIssuer:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF
```

## 6. Create the Limited User (CSR Flow)

```bash
# Generate private key and CSR
openssl genrsa -out nginx-deployer.key 2048
openssl req -new -key nginx-deployer.key -out nginx-deployer.csr -subj "/CN=nginx-deployer"
```

```bash
# Create the CertificateSigningRequest
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: nginx-deployer
spec:
  request: $(cat nginx-deployer.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
  - client auth
EOF
```

```bash
# Approve the CSR and extract the certificate
kubectl certificate approve nginx-deployer
kubectl get csr nginx-deployer -o jsonpath='{.status.certificate}' | base64 -d > nginx-deployer.crt
```

Then build the kubeconfig for the `nginx-deployer` user (see `manifests/limited-user/README.md` for the full steps).

## 7. Deploy the Application with Helm (as the limited user)

```bash
helm upgrade --install nginx-demo charts/nginx-demo \
  --namespace nginx-demo \
  --kubeconfig=$HOME/nginx-deployer.kubeconfig
```

---


