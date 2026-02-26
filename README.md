# 1.1 Update System & Install Dependencies
```
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg
```
# 1.2 Disable Swap (Required for K8s)
```
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab
```
# 1.3 Enable Kernel Modules
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```
# 1.4 Networking Setup (Sysctl)
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```
# 1.5 Install Containerd (Runtime)
```
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
# ตั้งค่า SystemdCgroup เป็น true (สำคัญมากสำหรับ Ubuntu 24.04)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```
# 2.1 Add Kubernetes Repository (v1.31)
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
# 2.2 Install Tools
```
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
# 3.1 Initialize Cluster
```
sudo kubeadm init --control-plane-endpoint "k8s-master-01" --pod-network-cidr=10.244.0.0/16
```
# 3.2 Setup Kubeconfig
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
# 3.3 Install Networking (Flannel)
```
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
# 3.4 Taint Masters (บังคับ Pod ทั่วไปไปรันที่ Worker)
```
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule --overwrite
```
# 4.1 Install Ingress Controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```
# 4.2 Patch to HostNetwork & Node IP Reporting
# ทำให้ ADDRESS ใน Ingress แสดงผลเป็น IP ของ Node เครื่องที่ Pod รันอยู่
```
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type json -p='[
  {"op": "replace", "path": "/spec/template/spec/hostNetwork", "value": true},
  {"op": "replace", "path": "/spec/template/spec/containers/0/args", 
   "value": [
     "/nginx-ingress-controller",
     "--election-id=ingress-nginx-leader",
     "--controller-class=k8s.io/ingress-nginx",
     "--ingress-class=nginx",
     "--configmap=\$(POD_NAMESPACE)/ingress-nginx-controller",
     "--report-node-internal-ip-address"
   ]}
]'
```
