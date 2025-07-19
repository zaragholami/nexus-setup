- **Step 1: Nexus Server Setup**

**1.1 Install Docker on VPS**

```
sudo apt update  
sudo apt install -y docker.io docker-compose  
sudo systemctl enable --now docker
```
**1.2 Deploy Nexus**

```
version: '3'  
services:  
  nexus:  
    image: sonatype/nexus3  
    ports:  
      - "8081:8081"  
      - "8082:8082"  
      - "8083:8083"  
    volumes:  
      - nexus-data:/nexus-data  
    environment:  
      - INSTALL4J_ADD_VM_PARAMS=-Xms512m -Xmx1024m  
volumes:  
  nexus-data:
```

```
docker-compose up -d
```
**1.3 Initialize Nexus**

1.Access http://<VPS_IP>:8081

2.Get admin password: docker exec -it nexus_nexus_1 cat /nexus-data/admin.password

3.Create repositories:

## 🧩 Nexus Proxy Configuration

Below are the configured proxy repositories in Nexus Repository Manager used to cache and serve different ecosystems:

| Type             | Name         | Upstream URL                          | Nexus Port |
|------------------|--------------|----------------------------------------|------------|
| **APT (proxy)**   | `apt-proxy`  | http://archive.ubuntu.com             | `8081`     |
| **YUM (proxy)**   | `yum-proxy`  | https://mirror.centos.org             | `8081`     |
| **Docker (proxy)**| `docker-proxy`| https://registry-1.docker.io          | `8082`     |
| **Docker (proxy)**| `k8s-proxy`  | https://registry.k8s.io               | `8083`     |
| **Helm (proxy)**  | `helm-proxy` | https://charts.helm.sh/stable         | `8081`     |

> ℹ️ All ports refer to the HTTP connector defined in Nexus. You can expose them via ingress, reverse proxy, or port-forwarding as needed.

- **Step 2: Client Configuration**

**2.1 APT Clients (Debian/Ubuntu)**
```
# /etc/apt/sources.list.d/nexus.list  
deb [trusted=yes] http://<NEXUS_IP>:8081/repository/apt-proxy/ focal main restricted
```
**2.2 YUM Clients (CentOS/RHEL)**

```
# /etc/yum.repos.d/nexus.repo  
[nexus-baseos]  
name=Nexus BaseOS  
baseurl=http://<NEXUS_IP>:8081/repository/yum-proxy/$releasever/BaseOS/$basearch/os/  
enabled=1  
gpgcheck=0
```
**2.3 Docker Clients**

```
# /etc/docker/daemon.json  
{  
  "registry-mirrors": ["http://<NEXUS_IP>:8082"],  
  "insecure-registries": ["<NEXUS_IP>:8082"]  
}
```
**2.4 Kubernetes Components**

```
# /etc/apt/sources.list.d/kubernetes.list (Debian/Ubuntu)  
deb http://<NEXUS_IP>:8081/repository/apt-proxy/ kubernetes-xenial main
```
**For Kubernetes images:**
```toml
# /etc/containerd/config.toml  
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]  
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]  
    endpoint = ["http://<NEXUS_IP>:8082"]  
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]  
    endpoint = ["http://<NEXUS_IP>:8083"]
```
- **Step 3: Kubernetes Installation Workflow**

**3.1 Install Kubernetes Components**

```
# On all nodes  
sudo apt update  
sudo apt install -y kubelet kubeadm kubectl
```
**3.2 Initialize Control Plane**
```
sudo kubeadm init \  
  --image-repository="<NEXUS_IP>:8083" \  
  --pod-network-cidr=10.244.0.0/16
```
**3.3 Join Worker Nodes**
```
sudo kubeadm join <control-plane-host>:<port> \  
  --token <token> \  
  --discovery-token-ca-cert-hash sha256:<hash> \  
  --image-repository="<NEXUS_IP>:8083"
```
**3.4 Verify Cluster**
```
kubectl get nodes  
kubectl get pods --all-namespaces
```
**Step 4: Maintenance & Updates**

**4.1 Pre-Cache Critical Packages**

```
# Docker images  
docker pull <NEXUS_IP>:8082/ubuntu:22.04  
docker pull <NEXUS_IP>:8083/kube-apiserver:v1.27.2  

# APT packages  
docker run --rm apt-cacher \  
  wget http://<NEXUS_IP>:8081/repository/apt-proxy/pool/main/u/ubuntu-keyring/ubuntu-keyring_2021.03.26_all.deb  

# Helm charts  
helm fetch nexus/nginx-ingress --version=4.0.13
```
**4.2 Update Workflow**

1.APT/YUM: Clients run normal update commands
```
sudo apt update && sudo apt upgrade  
sudo yum update
```
2.Kubernetes:
```
# Drain node  
kubectl drain <node> --ignore-daemonsets  

# Upgrade kubeadm  
sudo apt install kubeadm=<version>  

# Upgrade node  
sudo kubeadm upgrade node  

# Upgrade kubelet  
sudo apt install kubelet=<version>  
sudo systemctl restart kubelet  

# Uncordon node  
kubectl uncordon <node>
```
3.Docker Images:
```
docker pull <NEXUS_IP>:8082/<image>:<new-tag>
```
