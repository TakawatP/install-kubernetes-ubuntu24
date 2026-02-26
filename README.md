# 1. OS Preparation (ทุก Node)
```
# 1.1 Update ระบบและลง Dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg

# 1.2 ปิด Swap (สำคัญมาก)
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# 1.3 ตั้งค่า Kernel Modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# 1.4 ตั้งค่า Network (Sysctl)
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system

# 1.5 ติดตั้ง Containerd (Runtime)
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
# สำคัญ: Ubuntu 24.04 ต้องใช้ SystemdCgroup = true
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
sudo systemctl restart containerd
```
# 2. K8s Installation (ทุก Node)
```
# 2.1 เพิ่ม Repository (v1.31)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 2.2 ติดตั้ง Kubeadm, Kubelet, Kubectl
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
# 3. Cluster Setup (Master-01 เท่านั้น)
```
# 3.1 Initialize Cluster (ปรับ endpoint ตามชื่อเครื่อง)
sudo kubeadm init --control-plane-endpoint "k8s-master-01" --pod-network-cidr=10.244.0.0/16

# 3.2 ตั้งค่า Config สำหรับ User
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 3.3 ติดตั้ง Network Plugin (Flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# 3.4 ใส่ Taint ให้ Master (ห้าม Pod ทั่วไปมารันบนเครื่อง Master)
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule --overwrite
```
# 4. Ingress Controller Setup (Host Network Mode)
```
# 4.1 ติดตั้ง Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# 4.2 Patch ให้ใช้ Network เครื่อง และรายงาน Node IP
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type json -p='[
  {"op": "replace", "path": "/spec/template/spec/hostNetwork", "value": true},
  {"op": "replace", "path": "/spec/template/spec/containers/0/args", 
   "value": [
     "/nginx-ingress-controller",
     "--election-id=ingress-nginx-leader",
     "--controller-class=k8s.io/ingress-nginx",
     "--ingress-class=nginx",
     "--configmap=$(POD_NAMESPACE)/ingress-nginx-controller",
     "--report-node-internal-ip-address"
   ]}
]'
```
# 5. Critical Fix: Admission Webhook (ทำทันทีหลังลง Ingress)
```
# 5.1 ลบ Validation Webhook Configuration
kubectl delete validatingwebhookconfigurations ingress-nginx-admission

# 5.2 ลบการเรียกใช้ Volume/Secret ที่ค้างคาใน Deployment
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type json -p='[
  {"op": "remove", "path": "/spec/template/spec/containers/0/volumeMounts/0"},
  {"op": "remove", "path": "/spec/template/spec/volumes/0"}
]'
```
