To use **MetalLB** with **Ingress NGINX** in **K3d**, you need to configure both components correctly. Below is a step-by-step guide to set up MetalLB and Ingress NGINX in a K3d cluster:

---

### **1. Prerequisites**
- **K3d installed**: Ensure K3d is installed on your machine.  
  Install it using:
  ```bash
  curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
  ```

- **kubectl installed**: You need `kubectl` to interact with the cluster.

---

### **2. Create a K3d Cluster**
You need to create a K3d cluster with a `--k3s-arg` to disable the default Traefik Ingress Controller (if you're using Ingress NGINX instead). Run the following:

```bash
k3d cluster create mycluster \
  --k3s-arg "--disable=traefik@server:0" \
  --servers 1 --agents 2 \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"
```

This will:
- Create a K3d cluster named `mycluster`.
- Disable Traefik.
- Expose ports 80 and 443 on the host for external access.

---

### **3. Install MetalLB**
MetalLB is used to provide external IP addresses for the Ingress Controller.

#### a. **Install MetalLB**
Apply the MetalLB manifests:
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/manifests/metallb.yaml
```

#### b. **Configure MetalLB**
Create a configuration file for MetalLB with an IP range for LoadBalancer services. Use a range compatible with your Docker network (e.g., `172.18.0.100-172.18.0.200` for default K3d setups).

```yaml
# metallb-config.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-ip-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.0.100-172.18.0.200
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: my-l2-advertisement
  namespace: metallb-system
```

Apply the configuration:
```bash
kubectl apply -f metallb-config.yaml
```

---

### **4. Install Ingress NGINX**
Deploy the NGINX Ingress Controller to the K3d cluster.

#### a. **Add the Ingress NGINX Helm Repository**
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

#### b. **Install the NGINX Ingress Controller**
```bash
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

This will:
- Deploy the NGINX Ingress Controller.
- Create a LoadBalancer service that MetalLB will assign an external IP to.

#### c. **Verify the Ingress Controller**
Check the service for the external IP:
```bash
kubectl get svc -n ingress-nginx
```

You should see something like this:
```
NAME                                 TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.43.20.11      172.18.0.100     80:30080/TCP,443:30443/TCP   2m
ingress-nginx-controller-admission   ClusterIP      10.43.15.195     <none>           443/TCP                      2m
```

Note the `EXTERNAL-IP` (e.g., `172.18.0.100`).

---

### **5. Deploy an Application**
Deploy a sample app to test the Ingress.

#### a. **Deploy a Sample Application**
```yaml
# sample-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  labels:
    app: hello-world
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: hashicorp/http-echo
        args:
        - "-text=Hello, World!"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5678
  type: ClusterIP
```

Apply it:
```bash
kubectl apply -f sample-app.yaml
```

#### b. **Create an Ingress Resource**
```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
```

Apply it:
```bash
kubectl apply -f ingress.yaml
```

---

### **6. Test the Setup**
1. **Update `/etc/hosts`**:
   Add the external IP of the Ingress Controller and map it to the hostname:
   ```
   172.18.0.100 example.local
   ```

2. **Access the Application**:
   Open a browser and go to `http://example.local`. You should see:
   ```
   Hello, World!
   ```

---

### Summary of Components
- **MetalLB**: Assigns external IPs for `LoadBalancer` services.
- **Ingress NGINX**: Handles ingress traffic and routes it to backend services.
- **K3d**: Provides a lightweight Kubernetes environment.

Let me know if you face any issues or need help customizing this setup!
