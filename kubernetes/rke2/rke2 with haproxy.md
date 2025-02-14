Setting up an **RKE2 (Rancher Kubernetes Engine 2) High Availability (HA) Cluster** ensures reliability by running multiple control plane nodes and worker nodes. Below is a step-by-step guide to deploying an **HA RKE2 cluster with an embedded etcd database**.

---

## 🔹 **Architecture Overview**
- **At least 3 control plane nodes** (`server` nodes) with embedded etcd.
- **At least 2 worker nodes** (`agent` nodes).
- **A load balancer** (e.g., Nginx, HAProxy, Keepalived, or cloud LB) to distribute traffic to control plane nodes.

### 🖥️ **Example Nodes**
| Role          | Node Name   | IP Address     |
|--------------|------------|---------------|
| Load Balancer | lb01       | `192.168.1.10` |
| Control Plane | master01   | `192.168.1.11` |
| Control Plane | master02   | `192.168.1.12` |
| Control Plane | master03   | `192.168.1.13` |
| Worker Node   | worker01   | `192.168.1.21` |
| Worker Node   | worker02   | `192.168.1.22` |

---

## 🔹 **Step 1: Set Up a Load Balancer**
The load balancer distributes requests among control plane nodes. You can use **HAProxy** or **Nginx**.

### 🏗 **Example HAProxy Config (`/etc/haproxy/haproxy.cfg`)**
```txt
frontend rke2-lb
    bind *:6443
    mode tcp
    default_backend rke2-backend

backend rke2-backend
    mode tcp
    balance roundrobin
    server master01 192.168.1.11:6443 check
    server master02 192.168.1.12:6443 check
    server master03 192.168.1.13:6443 check
```
- Restart HAProxy:
  ```sh
  systemctl restart haproxy
  ```

---

## 🔹 **Step 2: Install RKE2 on Control Plane Nodes**
### 🏗 **Install RKE2 Binary on All Master Nodes**
Run this on each **control plane node** (`master01`, `master02`, `master03`):
```sh
curl -sfL https://get.rke2.io | sh -
```
Enable and start RKE2:
```sh
systemctl enable --now rke2-server
```
Check logs:
```sh
journalctl -u rke2-server -f
```

### **Configure the First Master Node (`master01`)**
1️⃣ **Create a Configuration File**:  
Edit `/etc/rancher/rke2/config.yaml`:
```yaml
token: "MySuperSecureToken"
tls-san:
  - "192.168.1.10"  # Load Balancer IP
  - "rke2.example.com"
disable:
  - rke2-ingress
node-label:
  - "node-role.kubernetes.io/control-plane=true"
```

2️⃣ **Restart RKE2**:
```sh
systemctl restart rke2-server
```

3️⃣ **Get the Cluster Token**:
```sh
cat /var/lib/rancher/rke2/server/node-token
```
Copy this token. You'll use it to join other control plane and worker nodes.

4️⃣ **Get the Cluster Join Address**:
```sh
kubectl get nodes
```

---

## 🔹 **Step 3: Join Additional Control Plane Nodes**
On `master02` and `master03`, configure RKE2 to join `master01`:

1️⃣ **Edit `/etc/rancher/rke2/config.yaml`**:
```yaml
server: "https://192.168.1.10:6443"  # Load Balancer IP
token: "MySuperSecureToken"
node-label:
  - "node-role.kubernetes.io/control-plane=true"
```

2️⃣ **Restart RKE2 Service**:
```sh
systemctl enable --now rke2-server
```

3️⃣ **Verify Control Plane Nodes**:
```sh
kubectl get nodes
```
All three control plane nodes should now be visible.

---

## 🔹 **Step 4: Join Worker Nodes**
On `worker01` and `worker02`, install the RKE2 agent:
```sh
curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE="agent" sh -
```

Create the agent config file at `/etc/rancher/rke2/config.yaml`:
```yaml
server: "https://192.168.1.10:6443"
token: "MySuperSecureToken"
```

Enable and start the RKE2 agent:
```sh
systemctl enable --now rke2-agent
```

Check if the worker nodes joined:
```sh
kubectl get nodes
```

---

## 🔹 **Step 5: Verify Cluster**
Check if everything is running:
```sh
kubectl get nodes
kubectl get pods -A
kubectl get svc -A
```

Check etcd status:
```sh
kubectl get pods -n kube-system | grep etcd
```

---

## ✅ **Final Notes**
- **Upgrade RKE2**:
  ```sh
  curl -sfL https://get.rke2.io | INSTALL_RKE2_VERSION=v1.28.4 sh -
  ```
- **Backup etcd**:
  ```sh
  rke2 etcd-snapshot --name my-backup
  ```
- **Check Cluster Health**:
  ```sh
  kubectl get componentstatuses
  ```

---

### 🔹 **Purpose of `/etc/rancher/rke2/config.yaml` in RKE2**  

The **`/etc/rancher/rke2/config.yaml`** file is the main **configuration file** for RKE2. It allows you to **customize** how RKE2 runs on each node (control plane or worker).  

Instead of passing long command-line arguments every time you start RKE2, you can define settings **once** in this YAML file.  

---

## 🔹 **Key Use Cases of `config.yaml`**
1. **Set Cluster Token** (For secure node registration)
2. **Define Server or Agent Mode**  
3. **Enable High Availability (HA)**  
4. **Specify Custom Ports, Networking, and Ingress**  
5. **Disable Unwanted Components (e.g., Traefik, metrics-server)**  
6. **Customize Certificates (TLS SANs)**  

---

## 🔹 **Common `config.yaml` Configurations**
### **1️⃣ Control Plane (`server` mode)**
If the node is a **control plane**, use:
```yaml
token: "MySuperSecureToken"  # Shared token for joining nodes
tls-san:
  - "192.168.1.10"  # Load Balancer IP
  - "rke2.example.com"
disable:
  - rke2-ingress
node-label:
  - "node-role.kubernetes.io/control-plane=true"
```
**💡 Why?**  
- Ensures control plane nodes accept connections via the **load balancer (`tls-san`)**.
- Disables built-in ingress if you're using another one.

---

### **2️⃣ Additional Control Plane Nodes (Joining the Cluster)**
For the **second and third control plane nodes**, use:
```yaml
server: "https://192.168.1.10:6443"  # Load balancer forwarding to control planes
token: "MySuperSecureToken"
```
**💡 Why?**  
- This tells new nodes **where to find the primary control plane**.

---

### **3️⃣ Worker Nodes (`agent` mode)**
For worker nodes, use:
```yaml
server: "https://192.168.1.10:6443"  # Connects to the API server via Load Balancer
token: "MySuperSecureToken"
```
**💡 Why?**  
- Workers **register** with the cluster and receive workloads.

---

### **4️⃣ Disable Unwanted Components**
By default, RKE2 installs some components like **Traefik and metrics-server**.  
You can disable them:
```yaml
disable:
  - rke2-ingress
  - metrics-server
  - servicelb
```

---

### **5️⃣ Custom Container Runtime (Disable Built-in Containerd)**
If you want to use **Docker instead of Containerd**:
```yaml
container-runtime-endpoint: "unix:///run/containerd/containerd.sock"
```

---

## 🔹 **How to Apply Configurations?**
After editing **`/etc/rancher/rke2/config.yaml`**, restart the service:
```sh
systemctl restart rke2-server  # For control plane
systemctl restart rke2-agent   # For worker nodes
```
Verify settings:
```sh
kubectl get nodes
```

---

## ✅ **Summary**
- **`config.yaml` simplifies RKE2 setup** by allowing you to define settings **once**.
- **Control plane nodes** use it to configure **TLS, HA, and API access**.
- **Worker nodes** use it to **join the cluster**.
- Helps with **disabling unwanted components** (like Traefik).
- Any changes require **restarting RKE2** to apply.

---

The differences between the **primary control plane configuration** and **additional control planes**, and also understand the role of **`tls-san`** in RKE2.

---

## 🔹 **Primary vs. Additional Control Plane Configurations**
| **Feature**       | **Primary Control Plane** | **Additional Control Planes** |
|------------------|------------------------|---------------------------|
| **Config File**  | `/etc/rancher/rke2/config.yaml` | `/etc/rancher/rke2/config.yaml` |
| **Defines Cluster Token?** | ✅ Yes | ❌ No (Just uses the token) |
| **Sets `server:` field?** | ❌ No | ✅ Yes (Points to Load Balancer) |
| **Uses `tls-san`?** | ✅ Yes (For LB & Custom DNS) | ❌ No (Uses API discovery) |
| **Runs `rke2-server`?** | ✅ Yes | ✅ Yes |
| **Runs etcd?** | ✅ Yes (etcd cluster initialized) | ✅ Yes (Joins etcd cluster) |

---

## 🔹 **Primary Control Plane (`config.yaml`)**
The **first control plane node** is responsible for **bootstrapping the cluster** and **initializing etcd**.  
It does **not** specify `server:` because it’s the one creating the cluster.

```yaml
token: "MySuperSecureToken"
tls-san:
  - "192.168.1.10"  # Load Balancer IP
  - "rke2.example.com"  # Custom DNS for API access
disable:
  - rke2-ingress
node-label:
  - "node-role.kubernetes.io/control-plane=true"
```
### **Why No `server:` Field?**
- The **first control plane node is the cluster itself** (it does not need to join another node).
- Other control planes will join **this** node.

### **Why Add `tls-san`?**
- **TLS-SAN (Subject Alternative Name) is used for SSL certificates**.
- By default, RKE2 **only issues a certificate for its real IP**.
- When you add **a Load Balancer IP (`192.168.1.10`) or domain (`rke2.example.com`)**, the API server can accept requests from those sources **without TLS errors**.

> 🛑 **Without `tls-san`, API requests through the Load Balancer would fail due to certificate mismatches!**  

---

## 🔹 **Additional Control Planes (`config.yaml`)**
For the **second and third control plane nodes**, they **must join the cluster** through the **load balancer**.

```yaml
server: "https://192.168.1.10:6443"  # Load Balancer IP
token: "MySuperSecureToken"
disable:
  - rke2-ingress
```
### **Why Do They Specify `server:`?**
- They **must connect to an existing API server** to join the cluster.
- The **Load Balancer distributes requests** to healthy control planes.

### **Why No `tls-san`?**
- `tls-san` is only needed on the **first node** to issue the correct API certificate.
- These nodes **use the existing certificate from the primary control plane**.

---

## 🔹 **How `tls-san` Helps with High Availability**
Imagine you **lose the first control plane node** (`192.168.1.11`).  
- If all API requests were pointing **only** to `192.168.1.11`, **Kubernetes would break**.
- Because we set `tls-san` to **`192.168.1.10` (Load Balancer)**, clients can still reach the cluster **through another control plane node**.

---

## ✅ **Final Summary**
1. **Primary control plane:**
   - **Bootstraps the cluster**.
   - Does **not** have `server:` because it is the first node.
   - Uses `tls-san` for **Load Balancer and Custom DNS**.

2. **Additional control planes:**
   - Must **join an existing cluster** (via `server: <LoadBalancerIP>`).
   - Do **not** need `tls-san` because they use **existing certificates**.

3. **`tls-san` is critical for high availability**:
   - Ensures the API server **accepts connections from the Load Balancer**.
   - Prevents TLS certificate mismatches when using **custom domain names**.

Hope that clears things up! Let me know if you have more questions. 🚀
