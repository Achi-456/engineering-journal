# Kubernetes Metrics Server Installation & Troubleshooting Guide

This guide details the establishment of the **Metrics Server v0.8.0** on a Kubernetes v1.35 cluster running on **RHEL 9.1** using **cri-dockerd**.

---

## 1. Installation Steps

### Phase A: Image Preparation (The Workaround)

Due to connectivity issues with `registry.k8s.io` on RHEL 9 nodes, the image must be manually pulled from the `k8s.gcr.io` mirror and tagged locally.

**Run on all nodes (Master & Workers):**

```bash
# Pull the image from the mirror
sudo docker pull k8s.gcr.io/metrics-server/metrics-server:v0.8.0

# Tag it to match the official manifest reference
sudo docker tag k8s.gcr.io/metrics-server/metrics-server:v0.8.0 registry.k8s.io/metrics-server/metrics-server:v0.8.0
```

---

### Phase B: Deployment via Ansible

Create `install-metrics.yml` on your Jumphost. Note the use of the `insecure-tls` flag required for `kubeadm` self-signed certificates.

```yaml
---
- name: Install Kubernetes Metrics Server
  hosts: control_plane
  become: no
  tasks:
    - name: Download Metrics Server manifest
      get_url:
        url: https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
        dest: /tmp/metrics-server.yaml

    - name: Patch Metrics Server for self-signed certs
      lineinfile:
        path: /tmp/metrics-server.yaml
        insertafter: '- args:'
        line: '        - --kubelet-insecure-tls'

    - name: Apply Metrics Server manifest
      command: kubectl apply -f /tmp/metrics-server.yaml
      environment:
        KUBECONFIG: /home/kube-deploy/.kube/config
```

**Execute the playbook:**

```bash
ansible-playbook -i hosts.ini install-metrics.yml -K
```

---

## 2. Troubleshooting & Common Reasons for Failure

### Error 1: `ErrImagePull` or `ImagePullBackOff`

**Reason:** The node cannot reach `registry.k8s.io`. This is common in RHEL 9 environments with restrictive firewalls or missing DNS entries in `/etc/resolv.conf`.

**Fix:** Manually pull the image from `k8s.gcr.io` or `docker.io/bitnami/metrics-server` and use `docker tag` to alias it to the name required by the manifest.

---

### Error 2: `dial tcp [::1]:8080: connect: connection refused`

**Reason:** `kubectl` is being run by a user (or Ansible task) that does not have the `KUBECONFIG` environment variable set. It defaults to trying to reach an unauthenticated API on localhost.

**Fix:** Ensure the `KUBECONFIG` environment variable points to `/home/kube-deploy/.kube/config` in your shell or Ansible task.

---

### Error 3: `Metrics API not available`

**Reason:** The Metrics Server pod is still starting, or it cannot verify the Kubelet certificates.

**Fix:** 
1. Wait 60–90 seconds after deployment.
2. Verify the `--kubelet-insecure-tls` argument is present in the Deployment `args` section.

---

### Error 4: YAML Syntax Error (Tabs vs Spaces)

**Reason:** Ansible/YAML parsers do not support **Tab** characters.

**Fix:** Use 2 or 4 spaces for indentation. Run `sed -i 's/\t/  /g' filename.yml` to convert tabs to spaces automatically.

---

## 3. Post-Installation Verification

Once established, use these commands to verify health and resource usage:

| Command | Purpose |
|---------|---------|
| `kubectl get pods -n kube-system -l k8s-app=metrics-server` | Check if pod is **1/1 Running** |
| `kubectl top nodes` | View CPU and RAM usage per Node |
| `kubectl top pods -A --sort-by=memory` | View hungriest Pods across all namespaces |
| `kubectl describe apiservice v1beta1.metrics.k8s.io` | Check the health of the Metrics API link |

---

## 4. Additional Notes

### Environment Details
- **Kubernetes Version:** v1.35
- **Operating System:** RHEL 9.1
- **Container Runtime:** cri-dockerd
- **Metrics Server Version:** v0.8.0

### Alternative Image Sources

If `k8s.gcr.io` is also unavailable, consider these alternatives:

```bash
# Option 1: Docker Hub mirror
sudo docker pull docker.io/bitnami/metrics-server:0.8.0
sudo docker tag docker.io/bitnami/metrics-server:0.8.0 registry.k8s.io/metrics-server/metrics-server:v0.8.0

# Option 2: Quay.io mirror (if available)
sudo docker pull quay.io/kubernetes/metrics-server:v0.8.0
sudo docker tag quay.io/kubernetes/metrics-server:v0.8.0 registry.k8s.io/metrics-server/metrics-server:v0.8.0
```

### Security Considerations

**Warning:** The `--kubelet-insecure-tls` flag disables TLS certificate verification between the Metrics Server and Kubelets. This is acceptable for development/testing environments but should be avoided in production.

**For production environments:**
1. Generate proper TLS certificates for your Kubelets
2. Configure the Metrics Server to use certificate verification
3. Remove the `--kubelet-insecure-tls` flag

---

## 5. Manual Deployment (Without Ansible)

If you prefer to deploy manually without Ansible:

```bash
# Download the manifest
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -O metrics-server.yaml

# Edit the file to add the insecure-tls flag
# Find the 'args:' section under the Deployment and add:
#   - --kubelet-insecure-tls

# Apply the manifest
kubectl apply -f metrics-server.yaml

# Wait for deployment to complete
kubectl rollout status deployment metrics-server -n kube-system
```

---

## 6. Cleanup (If Needed)

To remove the Metrics Server:

```bash
kubectl delete -f /tmp/metrics-server.yaml
```

To remove the Docker images:

```bash
sudo docker rmi registry.k8s.io/metrics-server/metrics-server:v0.8.0
sudo docker rmi k8s.gcr.io/metrics-server/metrics-server:v0.8.0
```

---

**Document Version:** 1.0  
**Last Updated:** January 2026  
**Tested On:** RHEL 9.1 | Kubernetes v1.35 | cri-dockerd
