## Kubernetes v1.31 Cluster Creation (RHEL 9.1, kubeadm + Docker)

### 1. VM layout (consistent addresses)
| Role | Hostname | IP Address | vCPU | RAM | Operating System (Image) |
| --- | --- | --- | --- | --- | --- |
| Control Plane | kubeadm-k8s-m1 | 172.25.199.161 | 2 | 4 GB | RHEL 9.1 (rhel-baseos-9.1-x86_64-dvd.iso) |
| Worker Node 1 | kubeadm-k8s-w1 | 172.25.199.162 | 4 | 8 GB | RHEL 9.1 (rhel-baseos-9.1-x86_64-dvd.iso) |
| Worker Node 2 | kubeadm-k8s-w2 | 172.25.199.163 | 4 | 8 GB | RHEL 9.1 (rhel-baseos-9.1-x86_64-dvd.iso) |
| Jumphost | k8s-mgmt-jh | 172.25.199.160 | 2 | 4 GB | RHEL 9.1 (rhel-baseos-9.1-x86_64-dvd.iso) |

### 2. SSH from jumphost (passwordless)
```bash
mkdir -p ~/.ssh

cat << EOF > ~/.ssh/config
Host master
    Hostname 172.25.199.161
    User kube-deploy

Host worker1
    Hostname 172.25.199.162
    User kube-deploy

Host worker2
    Hostname 172.25.199.163
    User kube-deploy
EOF

chmod 600 ~/.ssh/config
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
ssh-copy-id master
ssh-copy-id worker1
ssh-copy-id worker2
```

### 3. Ansible setup on jumphost
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

`prepare_nodes.yml`
```yaml
---
- name: Hardening OS for Kubernetes v1.31
  hosts: k8s_nodes
  become: yes
  tasks:
    - name: Ensure Hostname matches Inventory
      hostname:
        name: "{{ inventory_hostname }}"

    - name: Add host entries to /etc/hosts
      blockinfile:
        path: /etc/hosts
        block: |
          172.25.199.161 kubeadm-k8s-m1
          172.25.199.162 kubeadm-k8s-w1
          172.25.199.163 kubeadm-k8s-w2
          172.25.199.160 k8s-mgmt-jh

    - name: Create mount directory for ISO
      file:
        path: /mnt/iso
        state: directory
        mode: '0755'

    - name: Mount ISO
      mount:
        path: /mnt/iso
        src: /dev/cdrom
        fstype: iso9660
        state: mounted

    - name: Configure Local BaseOS Repository
      copy:
        dest: /etc/yum.repos.d/local-baseos.repo
        content: |
          [local-baseos]
          name=RHEL 9 Local - BaseOS
          baseurl=file:///mnt/iso/BaseOS
          enabled=1
          gpgcheck=0

    - name: Configure Local AppStream Repository
      copy:
        dest: /etc/yum.repos.d/local-appstream.repo
        content: |
          [local-appstream]
          name=RHEL 9 Local - AppStream
          baseurl=file:///mnt/iso/AppStream
          enabled=1
          gpgcheck=0

    - name: Clear DNF cache
      command: dnf clean all

    - name: Rebuild DNF cache
      command: dnf makecache

    - name: Disable Swap immediately
      command: swapoff -a
      when: ansible_swaptotal_mb > 0

    - name: Remove Swap from fstab (Permanent)
      replace:
        path: /etc/fstab
        regexp: '^([^#].*?\sswap\s+sw\s+.*)$'
        replace: '# \1'

    - name: Set SELinux to Permissive
      shell: |
        setenforce 0
        sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
      ignore_errors: yes

    - name: Load Kernel Modules for Container Networking
      modprobe:
        name: "{{ item }}"
        state: present
      loop:
        - overlay
        - br_netfilter

    - name: Persist Modules on Boot
      copy:
        dest: /etc/modules-load.d/k8s.conf
        content: |
          overlay
          br_netfilter

    - name: Ensure net.bridge.bridge-nf-call-iptables is set
      sysctl:
        name: net.bridge.bridge-nf-call-iptables
        value: '1'
        state: present
        reload: yes
        sysctl_file: /etc/sysctl.d/k8s.conf

    - name: Ensure net.bridge.bridge-nf-call-ip6tables is set
      sysctl:
        name: net.bridge.bridge-nf-call-ip6tables
        value: '1'
        state: present
        reload: yes
        sysctl_file: /etc/sysctl.d/k8s.conf

    - name: Ensure net.ipv4.ip_forward is set
      sysctl:
        name: net.ipv4.ip_forward
        value: '1'
        state: present
        reload: yes
        sysctl_file: /etc/sysctl.d/k8s.conf

    - name: Stop and Disable Firewalld
      command: systemctl disable --now firewalld
      ignore_errors: yes

    - name: Add Docker CE repository
      command: dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo

    - name: Install Docker packages
      yum:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present

    - name: Configure Docker to use systemd cgroups
      copy:
        dest: /etc/docker/daemon.json
        content: |
          {
            "exec-opts": ["native.cgroupdriver=systemd"]
          }
        mode: '0644'

    - name: Enable and start Docker
      systemd:
        name: docker
        enabled: yes
        state: started

    - name: Download cri-dockerd binary
      get_url:
        url: https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.21/cri-dockerd-0.3.21.amd64.tgz
        dest: /tmp/cri-dockerd-0.3.21.amd64.tgz

    - name: Extract cri-dockerd
      unarchive:
        src: /tmp/cri-dockerd-0.3.21.amd64.tgz
        dest: /tmp/
        remote_src: yes

    - name: Move cri-dockerd binary
      command: mv /tmp/cri-dockerd/cri-dockerd /usr/local/bin/

    - name: Download cri-dockerd systemd units
      get_url:
        url: "{{ item }}"
        dest: "/etc/systemd/system/{{ item | basename }}"
      loop:
        - https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service
        - https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket

    - name: Fix cri-dockerd binary path in service file
      replace:
        path: /etc/systemd/system/cri-docker.service
        regexp: '/usr/bin/cri-dockerd'
        replace: '/usr/local/bin/cri-dockerd'

    - name: Reload and start cri-dockerd
      systemd:
        daemon_reload: yes
        name: "{{ item }}"
        enabled: yes
        state: started
      loop:
        - cri-docker.socket
        - cri-docker.service
```

Run:
```bash
ansible-playbook -i hosts.ini prepare_nodes.yml -K
```

### 4. Install Kubernetes packages with Ansible
`kube-install.yml`
```yaml
---
- name: Install Kubernetes Packages (kubeadm, kubelet, kubectl)
  hosts: k8s_nodes
  become: yes
  tasks:
    - name: Create Kubernetes repository file
      copy:
        dest: /etc/yum.repos.d/kubernetes.repo
        content: |
          [kubernetes]
          name=Kubernetes
          baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
          enabled=1
          gpgcheck=1
          repo_gpgcheck=1
          gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
          exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni

    - name: Install Kubernetes components
      yum:
        name:
          - kubelet
          - kubeadm
          - kubectl
        state: present
        disable_excludes: kubernetes

    - name: Enable and Start kubelet
      systemd:
        name: kubelet
        enabled: yes
        state: started
```

Run:
```bash
ansible-playbook -i hosts.ini kube-install.yml -K
```

### 5. Control plane bootstrap (run on kubeadm-k8s-m1)
```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///var/run/cri-dockerd.sock \
  --ignore-preflight-errors=SystemVerification

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubeadm token create --print-join-command
# Append to the printed command: --cri-socket unix:///var/run/cri-dockerd.sock
```

### 6. Worker join (run on each worker)
If reusing nodes:
```bash
sudo kubeadm reset -f --cri-socket unix:///var/run/cri-dockerd.sock
```

Join (replace token/hash with current values):
```bash
sudo kubeadm join 172.25.199.161:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash <sha256:hash> \
  --cri-socket unix:///var/run/cri-dockerd.sock \
  --ignore-preflight-errors=SystemVerification
```

### 7. Flannel networking
On every node, prepare images (workaround for registry naming):
```bash
sudo docker pull docker.io/flannel/flannel:v0.27.4
sudo docker pull docker.io/flannel/flannel-cni-plugin:v1.8.0-flannel1
sudo docker tag docker.io/flannel/flannel:v0.27.4 ghcr.io/flannel-io/flannel:v0.27.4
sudo docker tag docker.io/flannel/flannel-cni-plugin:v1.8.0-flannel1 ghcr.io/flannel-io/flannel-cni-plugin:v1.8.0-flannel1
```

Optional: automate the pre-pull/tag with Ansible (run from the jumphost):
```bash
ansible-playbook -i hosts.ini prepare-images.yml -K
```

On the control plane:
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### 8. Final verification
From the control plane, confirm the cluster is healthy:
```bash
kubectl get nodes
kubectl get pods -A
```
