## Kubernetes v1.31 - Full Ansible Automation (with Jumphost)

This variant keeps the jumphost but drives every step through Ansible so the three Kubernetes nodes stay in sync. IPs and hostnames match the base guide:

| Role | Hostname | IP |
| --- | --- | --- |
| Control Plane | kubeadm-k8s-m1 | 172.25.199.161 |
| Worker 1 | kubeadm-k8s-w1 | 172.25.199.162 |
| Worker 2 | kubeadm-k8s-w2 | 172.25.199.163 |

### 1) Control node setup
```bash
sudo dnf install -y python3.9 python3-pip ansible-core
mkdir -p ~/k8s-deploy && cd ~/k8s-deploy
```

`hosts.ini`
```ini
[control_plane]
kubeadm-k8s-m1 ansible_host=172.25.199.161

[workers]
kubeadm-k8s-w1 ansible_host=172.25.199.162
kubeadm-k8s-w2 ansible_host=172.25.199.163

[k8s_nodes:children]
control_plane
workers

[all:vars]
ansible_user=kube-deploy
ansible_become=yes
ansible_become_method=sudo
```

### 2) Base OS prep
Re-use `prepare_nodes.yml` from the main guide and run:
```bash
ansible-playbook -i hosts.ini prepare_nodes.yml -K
```

### 3) Install Kubernetes packages
Run the existing `kube-install.yml` to place kubelet/kubeadm/kubectl on every node:
```bash
ansible-playbook -i hosts.ini kube-install.yml -K
```

### 4) Pre-pull and retag Flannel images (Ansible)
Use the optional optimization so every node has the images before bootstrap:
```bash
ansible-playbook -i hosts.ini prepare-images.yml -K
```

### 5) Bootstrap the control plane (Ansible ad-hoc)
```bash
ansible -i hosts.ini control_plane -b -m shell -a "\
kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///var/run/cri-dockerd.sock \
  --ignore-preflight-errors=SystemVerification"

# Copy admin.conf back to the jumphost for kubectl access
ansible -i hosts.ini control_plane -b -m fetch -a "src=/etc/kubernetes/admin.conf dest=~/k8s-deploy/admin.conf flat=yes"
export KUBECONFIG=~/k8s-deploy/admin.conf
```

Capture the join command from the init output (append `--cri-socket unix:///var/run/cri-dockerd.sock`).

### 6) Join workers (Ansible ad-hoc)
Replace token/hash with current values:
```bash
ansible -i hosts.ini workers -b -m shell -a "\
kubeadm join 172.25.199.161:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash <sha256:hash> \
  --cri-socket unix:///var/run/cri-dockerd.sock \
  --ignore-preflight-errors=SystemVerification"
```

### 7) Deploy Flannel CNI
```bash
ansible -i hosts.ini control_plane -b -m command -a "kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml"
```

### 8) Verify
```bash
ansible -i hosts.ini control_plane -b -m command -a "kubectl get nodes"
ansible -i hosts.ini control_plane -b -m command -a "kubectl get pods -A"
```
