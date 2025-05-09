Using Rancher with a K3d (K3s in Docker) cluster is a straightforward way to test and manage Kubernetes environments. Below is a step-by-step guide:

---

### **Prerequisites**
1. Install **Docker**: Ensure Docker is installed and running.
2. Install **k3d**: Follow the [official installation guide](https://k3d.io/#installation).
3. Install **kubectl**: Use the Kubernetes CLI to interact with your cluster.
4. Install **Helm** (optional): Useful for deploying Rancher via Helm charts.

---

### **Steps to Use Rancher with K3d**

#### **1. Create a K3d Cluster**
```bash
k3d cluster create mycluster --agents 2 --port "8080:80@loadbalancer" --port "8443:443@loadbalancer"
```
or
```bash
k3d cluster create rancher --api-port 6550 --servers 1 --agents 1 --k3s-arg "--disable=traefik@server:0" --port "80:80@loadbalancer" --port "443:443@loadbalancer" --api-port 0.0.0.0:6550 --wait
```
- `mycluster`: The name of your cluster.
- `--agents 2`: Creates 2 worker nodes.
- `--port`: Exposes ports for the Rancher UI (default: 80 and 443).

Verify the cluster:
```bash
kubectl cluster-info
```
Create nginx-ingress
```bash
helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx --namespace ingress-nginx --create-namespace
```

#### **2. Install Rancher**
Rancher can be installed using Helm. First, add the Rancher Helm repository:
```bash
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
helm repo update
```

Create a `cattle-system` namespace for Rancher:
```bash
kubectl create namespace cattle-system
```

Install a certificate manager for Rancher:
```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.11.1/cert-manager.yaml
```
Wait for the cert-manager to be ready:
```bash
kubectl get pods -n cert-manager
```

Install Rancher using Helm:
```bash
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher.localhost --set replicas=1
```

- `hostname=rancher.localhost`: The hostname for Rancher. You can replace it with your custom domain.
- `replicas=1`: Recommended for testing; use more replicas for production.

Verify Rancher deployment:
```bash
kubectl -n cattle-system rollout status deploy/rancher
```
If k3d running on nginx ingress, change rancher services from ClusterIP to NodePort

```bash
apiVersion: v1
kind: Service
metadata:
  name: rancher
  namespace: cattle-system
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30443
      protocol: TCP
      name: http
    - port: 8443
      targetPort: 443
      nodePort: 30444
      protocol: TCP
      name: https
  selector:
    app: rancher
```
To view nginx ingress pod controller

```bash
kubectl get pod -n cattle-system
```

#### **3. Access Rancher**
- Open `https://localhost:8443` in your browser.
- You might need to add a host entry (`127.0.0.1 rancher.localhost`) in your system's `/etc/hosts` file for the domain to resolve.
- Log in to Rancher. It will prompt you to set a password during the first login.

![image](https://github.com/user-attachments/assets/56ff8534-964c-4a57-9b1c-73491784e156)

![image](https://github.com/user-attachments/assets/d1188687-d7b2-4b78-9f34-a719b0e5abee)


---

### **Managing K3d Cluster with Rancher**
1. **Add K3d Cluster to Rancher**:
   - Rancher will automatically detect the K3d cluster where it's deployed.
   - Alternatively, you can import another K3d cluster using the Rancher UI by copying and applying the provided `kubectl` command.

2. **Work with Workloads**:
   - Use Rancher’s UI to deploy workloads, configure namespaces, and manage Kubernetes resources.

3. **Test Rancher Features**:
   - Explore features like user authentication, role-based access control (RBAC), and application catalogs.

---

### **Cleanup**
To remove the K3d cluster and Rancher:
```bash
k3d cluster delete mycluster
kubectl delete namespace cattle-system
kubectl delete namespace cert-manager
```

This setup is perfect for testing Rancher features locally. Let me know if you need further assistance!
