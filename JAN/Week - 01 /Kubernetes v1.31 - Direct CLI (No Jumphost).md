## Kubernetes v1.31 - Direct CLI Install (No Jumphost)

Run everything directly on the three nodes (no SSH relay). Addresses and hostnames stay the same:

| Role | Hostname | IP |
| --- | --- | --- |
| Control Plane | kubeadm-k8s-m1 | 172.25.199.161 |
| Worker 1 | kubeadm-k8s-w1 | 172.25.199.162 |
| Worker 2 | kubeadm-k8s-w2 | 172.25.199.163 |

### 1) Per-node system prep
Execute on **each** node:
```bash
sudo hostnamectl set-hostname <node-hostname>
sudo tee -a /etc/hosts >/dev/null <<'EOF'
172.25.199.161 kubeadm-k8s-m1
172.25.199.162 kubeadm-k8s-w1
172.25.199.163 kubeadm-k8s-w2
EOF
sudo swapoff -a && sudo sed -i '/ swap / s/^/#/' /etc/fstab
sudo setenforce 0 && sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
sudo modprobe overlay br_netfilter
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
cat <<'EOF' | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
sudo systemctl disable --now firewalld
```

### 2) Install Docker and cri-dockerd
```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
echo '{"exec-opts": ["native.cgroupdriver=systemd"]}' | sudo tee /etc/docker/daemon.json
sudo systemctl enable --now docker

curl -L -o /tmp/cri-dockerd.tgz https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.21/cri-dockerd-0.3.21.amd64.tgz
sudo tar -xf /tmp/cri-dockerd.tgz -C /tmp
sudo mv /tmp/cri-dockerd/cri-dockerd /usr/local/bin/
sudo curl -L -o /etc/systemd/system/cri-docker.service https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
sudo curl -L -o /etc/systemd/system/cri-docker.socket https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket
sudo sed -i 's#/usr/bin/cri-dockerd#/usr/local/bin/cri-dockerd#g' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable --now cri-docker.socket cri-docker.service
```

### 3) Install Kubernetes packages
```bash
cat <<'EOF' | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

### 4) Pre-pull Flannel images
Run on every node (manual path without Ansible):
```bash
sudo docker pull docker.io/flannel/flannel:v0.27.4
sudo docker pull docker.io/flannel/flannel-cni-plugin:v1.8.0-flannel1
sudo docker tag docker.io/flannel/flannel:v0.27.4 ghcr.io/flannel-io/flannel:v0.27.4
sudo docker tag docker.io/flannel/flannel-cni-plugin:v1.8.0-flannel1 ghcr.io/flannel-io/flannel-cni-plugin:v1.8.0-flannel1
```

### 5) Initialize control plane (on kubeadm-k8s-m1)
```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///var/run/cri-dockerd.sock \
  --ignore-preflight-errors=SystemVerification

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Record the join command printed (append `--cri-socket unix:///var/run/cri-dockerd.sock`).

### 6) Join workers (on each worker)
```bash
sudo kubeadm reset -f --cri-socket unix:///var/run/cri-dockerd.sock  # if reusing node
sudo kubeadm join 172.25.199.161:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash <sha256:hash> \
  --cri-socket unix:///var/run/cri-dockerd.sock \
  --ignore-preflight-errors=SystemVerification
```

### 7) Deploy Flannel CNI (control plane)
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### 8) Verify
```bash
kubectl get nodes
kubectl get pods -A
```
