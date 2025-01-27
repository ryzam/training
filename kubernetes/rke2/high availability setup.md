To set up a highly available (HA) RKE2 cluster with **3 master nodes** and **2 worker nodes**, and install Rancher server management inside the cluster, follow these steps:

---

## **1. Prerequisites**
1. **System Requirements**:
   - Master Nodes: Minimum 2 CPUs, 4GB RAM, 50GB storage.
   - Worker Nodes: Minimum 1 CPU, 2GB RAM, 20GB storage.
   - Supported OS: Ubuntu 22.04, CentOS 8, or other compatible Linux distributions.

2. **Networking**:
   - Ensure all nodes can communicate with each other over the network.
   - Disable swap: `swapoff -a` and comment the swap entry in `/etc/fstab`.
   - Configure unique hostnames for all nodes.
   - Open required ports for RKE2 and Rancher (e.g., 6443, 2379, 80, 443).

3. **Install Dependencies**:
   ```bash
   sudo apt update && sudo apt install -y curl wget socat conntrack ipset
   ```

4. **Load Balancer**:
   Set up a load balancer (e.g., HAProxy or NGINX) to balance traffic across the master nodes for the Kubernetes API. Example HAProxy configuration:
   ```haproxy
   frontend kubernetes_api
       bind *:6443
       mode tcp
       option tcplog
       default_backend kubernetes_masters

   backend kubernetes_masters
       mode tcp
       balance roundrobin
       server master1 <MASTER1_IP>:6443 check
       server master2 <MASTER2_IP>:6443 check
   ```

---

## **2. Install RKE2**
### **On All Master Nodes**
1. Install RKE2:
   ```bash
   curl -sfL https://get.rke2.io | sh -
   ```
2. Enable RKE2 server:
   ```bash
   sudo systemctl enable rke2-server.service
   ```
3. Configure RKE2 for HA:
   - Edit `/etc/rancher/rke2/config.yaml` on **master nodes**:
     ```yaml
     server: https://<LOAD_BALANCER_IP>:6443
     token: "<CLUSTER_SECRET_TOKEN>"
     tls-san:
       - "<LOAD_BALANCER_IP>"
     ```
   - Replace `<LOAD_BALANCER_IP>` with your load balancer's IP and `<CLUSTER_SECRET_TOKEN>` with a strong, unique token.
4. Start RKE2 server:
   ```bash
   sudo systemctl start rke2-server.service
   ```

### **On All Worker Nodes**
1. Install RKE2:
   ```bash
   curl -sfL https://get.rke2.io | sh -
   ```
2. Enable RKE2 agent:
   ```bash
   sudo systemctl enable rke2-agent.service
   ```
3. Configure the worker nodes:
   - Edit `/etc/rancher/rke2/config.yaml`:
     ```yaml
     server: https://<LOAD_BALANCER_IP>:6443
     token: "<CLUSTER_SECRET_TOKEN>"
     ```
   - Use the same `<LOAD_BALANCER_IP>` and `<CLUSTER_SECRET_TOKEN>` as the master nodes.
4. Start RKE2 agent:
   ```bash
   sudo systemctl start rke2-agent.service
   ```

---

## **3. Verify Cluster Setup**
1. On one master node, verify cluster status:
   ```bash
   sudo /var/lib/rancher/rke2/bin/kubectl get nodes
   ```
   Ensure all nodes are listed as `Ready`.

2. Save the `kubeconfig` for local access:
   ```bash
   mkdir -p ~/.kube
   sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
   sudo chown $(id -u):$(id -g) ~/.kube/config
   ```

---

## **4. Install Rancher Server**
1. Add the Helm repository:
   ```bash
   helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
   helm repo update
   ```

2. Install Cert-Manager:
   ```bash
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.3/cert-manager.yaml
   kubectl wait --for=condition=available --timeout=300s deployment/cert-manager -n cert-manager
   ```

3. Install Rancher:
   ```bash
   helm install rancher rancher-latest/rancher \
       --namespace cattle-system --create-namespace \
       --set hostname=<RANCHER_HOSTNAME> \
       --set replicas=2 \
       --set bootstrapPassword=admin
   ```
   Replace `<RANCHER_HOSTNAME>` with a valid domain name pointing to your cluster.

4. Verify Rancher installation:
   ```bash
   kubectl -n cattle-system rollout status deploy/rancher
   ```

5. Access Rancher:
   - Open a browser and navigate to `https://<RANCHER_HOSTNAME>`.
   - Log in with the default username `admin` and the password set during installation.

---

## **5. Optional: Set Up Persistent Storage**
Install a CSI driver like Longhorn or NFS to provide persistent storage for workloads in your cluster.

---

Need further details on any of the steps?
