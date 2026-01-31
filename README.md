# Multi-Node Kubernetes Cluster with Kubeadm 1.34 (containerd)

## Architecture 
<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/5c03eb3e-9ec6-4e48-b5f6-662c017d4440" />


https://kubernetes.io/docs/reference/networking/ports-and-protocols/

## 🔐 Kubernetes Control Plane – Network Ports
| Protocol | Direction | Port / Range  | Purpose                 | Used By                           |
| -------- | --------- | ------------- | ----------------------- | --------------------------------- |
| TCP      | Inbound   | **6443**      | Kubernetes API Server   | All (kubectl, nodes, controllers) |
| TCP      | Inbound   | **2379–2380** | etcd server client API  | kube-apiserver, etcd              |
| TCP      | Inbound   | **10250**     | Kubelet API             | Control plane, self               |
| TCP      | Inbound   | **10259**     | kube-scheduler          | Self                              |
| TCP      | Inbound   | **10257**     | kube-controller-manager | Self                              |
<img width="1626" height="603" alt="image" src="https://github.com/user-attachments/assets/17079c52-a196-473c-b553-d433216e0372" />




## 🖥️ Kubernetes Worker Nodes – Network Ports

| Protocol | Direction | Port / Range    | Purpose           | Used By              |
| -------- | --------- | --------------- | ----------------- | -------------------- |
| TCP      | Inbound   | **10250**       | Kubelet API       | Control plane        |
| TCP      | Inbound   | **10256**       | kube-proxy        | Self, Load Balancers |
| TCP      | Inbound   | **30000–32767** | NodePort Services | All                  |
| UDP      | Inbound   | **30000–32767** | NodePort Services | All                  |

<img width="1605" height="516" alt="image" src="https://github.com/user-attachments/assets/d27fd562-aae1-4a33-b1af-b9b799715fd9" />

---
<img width="1240" height="675" alt="image" src="https://github.com/user-attachments/assets/3906e1bb-0015-4973-9a7f-7837b6864478" />


---

# **Clean Latest Kubernetes Multi-Node Setup (Kubeadm + Calico)**

**Target:** Ubuntu 22.04, Kubernetes 1.35+, containerd runtime, Calico CNI.

---

## **Step 0: Prerequisites (AWS)**

**EC2 instances:**

| Node          | IP (Private) | Hostname     | Role   |
| ------------- | ------------ | ------------ | ------ |
| Control Plane | 172.31.43.1  | controlplane | Master |
| Worker 1      | 172.31.43.11 | worker1      | Worker |
| Worker 2      | 172.31.43.12 | worker2      | Worker |

**Security groups:**

* **Master SG**: allow **6443, 2379-2380, 10250, 10257, 10259, 22** from VPC or your IP.
* **Worker SG**: allow **10250, 10256, NodePort 30000-32767, 22** from VPC.
* Allow **TCP 179** between all nodes (for Calico BGP).

> Make sure **source/destination checks are disabled** on EC2 instances (important for Calico).

---

## **Step 1: Set Hostnames**

**Run on all nodes BEFORE kubeadm install:**

```bash
sudo hostnamectl set-hostname controlplane   # Master
sudo hostnamectl set-hostname worker1        # Worker1
sudo hostnamectl set-hostname worker2        # Worker2
```

**Update /etc/hosts on all nodes:**

```text
172.31.43.1   controlplane
172.31.43.11  worker1
172.31.43.12  worker2
```

---

## **Step 2: Update System & Install Dependencies**

**All nodes:**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release net-tools
```

---

## **Step 3: Disable Swap and Configure Kernel**

**All nodes:**

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

**Verify:**

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

---

## **Step 4: Install containerd**

**All nodes (clean method, latest stable):**

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

**Verify:**

```bash
containerd --version
sudo systemctl status containerd
```

---

## **Step 5: Install Kubernetes (kubeadm, kubelet, kubectl)**

**All nodes:**

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet
```

**Verify versions:**

```bash
kubeadm version
kubectl version --client
kubelet --version
```

---

## **Step 6: Initialize Control Plane**

**Master node ONLY:**

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<MASTER-PRIVATE-IP> \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket unix:///var/run/containerd/containerd.sock \
  --node-name controlplane
```

* Save the **`kubeadm join ...`** command output for workers.

**Setup kubeconfig:**

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## **Step 7: Install Calico CNI**

**Master ONLY:**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.3/manifests/calico.yaml
```

* Wait for all pods in `calico-system` namespace to become `Running`:

```bash
kubectl get pods -n calico-system -w
```

**If Calico fails**, check your interface:

```bash
kubectl set env daemonset/calico-node -n calico-system IP_AUTODETECTION_METHOD=interface=ens5
kubectl delete pod -n calico-system -l k8s-app=calico-node
```

---

## **Step 8: Join Worker Nodes**

**On each worker node:**

```bash
sudo kubeadm join 172.31.43.1:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

**Verify on master:**

```bash
kubectl get nodes
```

Expected output:

```
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   5m    v1.35.x
worker1        Ready    <none>          2m    v1.35.x
worker2        Ready    <none>          2m    v1.35.x
```

---

## **Step 9: Optional – Allow Scheduling on Master (Single Node Lab)**

**Master ONLY:**

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## **Step 10: Validation**

**Master ONLY:**

```bash
kubectl get nodes
kubectl get pods -A
kubectl describe node worker1
```

✅ All nodes should be `Ready` and Calico pods `Running`.

---

### **Important Notes / Common Issues**

1. **Do not mix multiple containerd installation methods** (manual tar vs apt). Stick to **apt install containerd**.
2. **Use a single Calico version**: v3.31.x is latest and works with kube 1.35+.
3. **Disable source/destination check** on EC2 instances (needed for Calico).
4. **Security Groups** must allow **TCP 179** for BGP and all required Kubernetes ports.
5. **Always set hostnames before `kubeadm init`**.

---

# Cleanup the failed attempt

```bash
sudo kubeadm reset -f
sudo rm -rf ~/.kube
# Retry init
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=[IP_ADDRESS] --node-name controlplane

```
# Prepared by:
*Shaik Moulali*


