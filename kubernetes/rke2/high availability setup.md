iTo set up a highly available (HA) RKE2 cluster with **3 master nodes** and **2 worker nodes**, and install Rancher server management inside the cluster, follow these steps:

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

Setting up **HAProxy** to load balance **Kubernetes RKE2 master nodes** is essential for a high-availability (HA) control plane. The goal is to ensure that API requests are distributed across multiple RKE2 server nodes.

### **1. Install HAProxy**
You need a dedicated load balancer node (or multiple if using multiple HAProxy instances). Install HAProxy on this node:

```sh
sudo apt update && sudo apt install -y haproxy
```

For **RHEL/CentOS**, use:

```sh
sudo yum install -y haproxy
```

---

### **2. Configure HAProxy**
Edit the HAProxy configuration file:

```sh
sudo nano /etc/haproxy/haproxy.cfg
```

#### **Example HAProxy Configuration for RKE2**
```cfg
global
    log stdout format raw local0
    maxconn 2000

defaults
    log global
    mode tcp
    retries 3
    timeout connect 5s
    timeout client 50s
    timeout server 50s

frontend rke2-api
    bind *:6443
    default_backend rke2-masters

backend rke2-masters
    balance roundrobin
    option httpchk GET /healthz
    http-check expect status 200
    server master1 192.168.1.10:6443 check
    server master2 192.168.1.11:6443 check
    server master3 192.168.1.12:6443 check
```

#### **Explanation**
- The **frontend `rke2-api`** listens on port **6443** (Kubernetes API).
- The **backend `rke2-masters`** distributes traffic across multiple RKE2 master nodes.
- **Health checks** ensure only healthy nodes receive traffic.

---

### **3. Enable and Restart HAProxy**
Save and exit the file, then restart HAProxy:

```sh
sudo systemctl enable haproxy
sudo systemctl restart haproxy
sudo systemctl status haproxy
```

---

### **4. Configure RKE2 Nodes to Use HAProxy**
On each RKE2 master and agent node, configure it to use the HAProxy **VIP (Virtual IP)** instead of individual masters.

Modify or create `/etc/rancher/rke2/config.yaml`:

```yaml
server: https://<haproxy-ip>:6443
token: "<your-rke2-token>"
```

Restart RKE2:

```sh
sudo systemctl restart rke2-server
```

---

### **5. Test the Setup**
Check if HAProxy is forwarding requests:

```sh
kubectl get nodes --kubeconfig /etc/rancher/rke2/rke2.yaml
```

## Troubleshot
If both `/readyz` and `/healthz` return **"Unauthorized"**, it means the Kubernetes API server in RKE2 is enforcing authentication even for health checks. This is different from standard Kubernetes behavior, but RKE2 sometimes does this when certain configurations are enabled.

### **1. Try Checking API Server Health with `kubectl` (On the Master Node)**
Run this directly on an **RKE2 master node**:
```sh
kubectl get --raw='/healthz'
```
✅ If it returns `ok`, then the API is up and running. The issue is just that HAProxy’s health check is unauthenticated.

---

### **2. Fix HAProxy Health Check Using TCP Mode**
Since RKE2’s API requires authentication, HAProxy's **HTTP health checks will fail**. Instead, we can switch to a simple **TCP check** to verify if the Kubernetes API server is responding.

#### **Update HAProxy Configuration**
Edit `/etc/haproxy/haproxy.cfg`:
```cfg
frontend rke2-api
    bind *:6443
    default_backend rke2-masters

backend rke2-masters
    balance roundrobin
    option ssl-hello-chk
    server master1 192.168.1.10:6443 check
    server master2 192.168.1.11:6443 check
    server master3 192.168.1.12:6443 check
```

#### **Why Use `option ssl-hello-chk`?**
- This performs an **SSL handshake** instead of sending an HTTP request.
- It works even when authentication is required because it only checks if the port is open.

#### **Restart HAProxy**
```sh
sudo systemctl restart haproxy
```

---

### **3. Verify HAProxy Connectivity to Masters**
On the HAProxy node, check if it can connect to the RKE2 masters:

```sh
nc -vz 192.168.1.10 6443
```
✅ Expected output:
```
Connection to 192.168.1.10 6443 port [tcp/*] succeeded!
```

If this fails, check:
- Firewall rules (`sudo ufw allow 6443/tcp` or `firewall-cmd --add-port=6443/tcp --permanent`)
- If the API server is running (`sudo systemctl status rke2-server`)

---

### **4. Test HAProxy Logs**
After restarting HAProxy, check if it's working:
```sh
sudo journalctl -u haproxy --no-pager | tail -n 50
```
If you no longer see **"no server available"**, the issue is resolved.

---

### **Summary**
✔ Use **`option ssl-hello-chk`** in HAProxy to perform a TCP-level health check.  
✔ Restart HAProxy and verify logs.  
✔ Confirm connectivity using `nc -vz <master-ip> 6443`.  

This should fix the **"Unauthorized"** issue while still ensuring that HAProxy properly load balances your RKE2 master nodes.

---

## **2. Install RKE2**
### **On All Master Nodes**
1. Install RKE2:
   ```bash
   curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=server sudo sh -
   ```

2. Enable RKE2 server:
   ```bash
   sudo systemctl enable rke2-server.service
   ```

3. Configure RKE2 for HA:
   - sudo mkdir -p /etc/rancher/rke2
   - sudo nano /etc/rancher/rke2/config.yaml
   - Edit `/etc/rancher/rke2/config.yaml` on **master nodes**:
     On the first master node
     ```yaml
     token: "<token>"
     tls-san:
      - server-mastername
      - server-masterip
     ```
  
     On the other master node
     ```yaml
     server: https://<LOAD_BALANCER_IP>:6443
     token: "<CLUSTER_SECRET_TOKEN>"
     tls-san:
       - "<LOAD_BALANCER_IP>"
     ```
   - Replace `<LOAD_BALANCER_IP>` with your load balancer's IP and `<CLUSTER_SECRET_TOKEN>` with a strong, unique token.

4. kubeconfig file will be written to /etc/rancher/rke2/rke2.yaml. We need to copy into .kube/config
   ```bash
   mkdir .kube
   cp /etc/rancher/rke2/rke2.yaml .kube/config
   ```

5. Start RKE2 server:
   ```bash
   sudo systemctl start rke2-server.service
   ```
6. View status
   - sudo systemctl status rke2-server.service
   - sudo journalctl -u rke2-server.service -f
  
7. Copy server node-token
   ```bash
   sudo cat /var/lib/rancher/rke2/server/node-token
   ```
     
### **On All Worker Nodes**
1. Install RKE2:
   ```bash
   curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=agent sh -
   ```
2. Enable RKE2 agent:
   ```bash
   sudo systemctl enable rke2-agent.service
   ```
3. Configure the worker nodes:
   - sudo mkdir -p /etc/rancher/rke2
   - sudo nano /etc/rancher/rke2/config.yaml
   - Edit `/etc/rancher/rke2/config.yaml`:
     ```yaml
     server: https://<LOAD_BALANCER_IP>:9345
     token: "<CLUSTER_SECRET_TOKEN>"
     ```
   - Use the same `<LOAD_BALANCER_IP>` and `<CLUSTER_SECRET_TOKEN>` as the master nodes.
5. Start RKE2 agent:
   ```bash
   sudo systemctl start rke2-agent.service
   ```
6. View status
   - sudo systemctl status rke2-agent.service
   - sudo journalctl -u rke2-agent.service -f
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

Here's how you can set up **HAProxy** on a dedicated node to load balance traffic across your RKE2 master nodes:

---

## **Step 1: Prepare the New Node**
1. **System Requirements**:
   - At least 1 CPU, 512MB RAM, and 10GB disk space.
   - Ensure the node has network access to the RKE2 master nodes.

2. **Install Prerequisites**:
   - Update the package list and install required tools:
     ```bash
     sudo apt update && sudo apt install -y curl vim
     ```

3. **Ensure Network Access**:
   - Verify the HAProxy node can reach all master nodes on port `6443`:
     ```bash
     curl -k https://<MASTER_NODE_IP>:6443
     ```
     Replace `<MASTER_NODE_IP>` with your RKE2 master nodes' IPs.

---

## **Step 2: Install HAProxy**
1. Install HAProxy using the package manager:
   ```bash
   sudo apt update && sudo apt install -y haproxy
   ```

2. Verify the installation:
   ```bash
   haproxy -v
   ```
   Example output: `HA-Proxy version 2.x.x`

---

## **Step 3: Configure HAProxy**
1. Backup the default configuration:
   ```bash
   sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.bak
   ```

2. Edit the configuration file:
   ```bash
   sudo vim /etc/haproxy/haproxy.cfg
   ```

3. Replace the contents of the file with the following configuration:
   ```haproxy
   global
       log /dev/log    local0
       log /dev/log    local1 notice
       chroot /var/lib/haproxy
       stats socket /run/haproxy/admin.sock mode 660 level admin
       stats timeout 30s
       user haproxy
       group haproxy
       daemon

   defaults
       log     global
       option  tcplog
       option  dontlognull
       timeout connect 5000ms
       timeout client  50000ms
       timeout server  50000ms
       errorfile 400 /etc/haproxy/errors/400.http
       errorfile 403 /etc/haproxy/errors/403.http
       errorfile 408 /etc/haproxy/errors/408.http
       errorfile 500 /etc/haproxy/errors/500.http
       errorfile 502 /etc/haproxy/errors/502.http
       errorfile 503 /etc/haproxy/errors/503.http
       errorfile 504 /etc/haproxy/errors/504.http

   frontend kubernetes_api
       bind *:6443
       mode tcp
       option tcplog
       default_backend kubernetes_masters

   backend kubernetes_masters
       mode tcp
       balance roundrobin
       option tcp-check
       server master1 <MASTER1_IP>:6443 check
       server master2 <MASTER2_IP>:6443 check
   ```
   - Replace `<MASTER1_IP>` and `<MASTER2_IP>` with the IPs of your master nodes.
   - The `bind *:6443` ensures HAProxy listens on port 6443 for incoming traffic.

4. Test the configuration:
   ```bash
   sudo haproxy -f /etc/haproxy/haproxy.cfg -c
   ```
   If the configuration is valid, it will display `Configuration file is valid`.

---

## **Step 4: Start and Enable HAProxy**
1. Restart the HAProxy service to apply changes:
   ```bash
   sudo systemctl restart haproxy
   ```

2. Enable HAProxy to start on boot:
   ```bash
   sudo systemctl enable haproxy
   ```

3. Verify HAProxy is running:
   ```bash
   sudo systemctl status haproxy
   ```

---

## **Step 5: Test HAProxy**
1. On another machine, test the load balancer:
   ```bash
   curl -k https://<HA_PROXY_NODE_IP>:6443
   ```
   Replace `<HA_PROXY_NODE_IP>` with the IP of your HAProxy node. You should get a Kubernetes API response.

2. To test failover, temporarily stop one master node:
   ```bash
   sudo systemctl stop rke2-server
   ```
   Then try the API request again. HAProxy should route traffic to the other master node.

---

## **Optional: Monitor HAProxy**
1. Enable HAProxy stats (for monitoring):
   Add the following section to `/etc/haproxy/haproxy.cfg`:
   ```haproxy
   listen stats
       bind *:8404
       stats enable
       stats uri /stats
       stats refresh 10s
   ```
2. Restart HAProxy:
   ```bash
   sudo systemctl restart haproxy
   ```
3. Access the stats page at `http://<HA_PROXY_NODE_IP>:8404/stats`.

---

