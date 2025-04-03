---

## **Corrected HAProxy Configuration**
Since **workers connect via port 6443** and **masters communicate on port 9345**, your **HAProxy** config should handle both ports.  

### **Edit `/etc/haproxy/haproxy.cfg`**
```plaintext
frontend rke2_api
    bind *:6443
    mode tcp
    option tcplog
    default_backend rke2_masters

frontend rke2_server
    bind *:9345
    mode tcp
    option tcplog
    default_backend rke2_masters

backend rke2_masters
    mode tcp
    balance roundrobin
    option tcp-check
    server master1 <master1-ip>:9345 check
    server master2 <master2-ip>:9345 check
    server master3 <master3-ip>:9345 check
```
🔹 **Explanation:**  
- `frontend rke2_api` → Listens on **port 6443** (for worker nodes and `kubectl` requests).  
- `frontend rke2_server` → Listens on **port 9345** (for control-plane internal communication).  
- `backend rke2_masters` → Forwards traffic to **port 9345** on the master nodes.  

---

## **Master Nodes Configuration**
On **master1** (initial master node):
```yaml
token: "my-shared-token"
tls-san:
  - "haproxy-ip"
  - "master1-ip"
  - "master2-ip"
  - "master3-ip"
cluster-init: true
```

On **master2 and master3**:
```yaml
token: "my-shared-token"
tls-san:
  - "haproxy-ip"
server: "https://haproxy-ip:9345"
```

---

## **Worker Nodes Configuration**
On **worker nodes** (`worker1`, `worker2`, `worker3`):
```yaml
token: "my-shared-token"
server: "https://haproxy-ip:6443"
```

---

## **Final Notes**
✅ **Masters use port 9345** (for control-plane communication).  
✅ **Workers use port 6443** (for API requests).  
✅ **HAProxy forwards both 6443 (workers) and 9345 (masters) to the correct destination.**  
