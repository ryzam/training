To create a **K3d** (lightweight k3s cluster in Docker) with an ingress controller, follow these steps:

---

## **1. Install k3d & kubectl**
If you haven’t installed **k3d**, install it using:

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

Ensure **kubectl** is installed as well:

```bash
sudo apt install -y kubectl  # Ubuntu/Debian
brew install kubectl         # macOS
```

---

## **2. Create a k3d Cluster with Ingress**
By default, K3s comes with **Traefik** as an ingress controller, but you can explicitly enable it.

```bash
k3d cluster create mycluster \
  --servers 1 \
  --agents 2 \
  --port "80:80@loadbalancer" \
  --port "443:443@loadbalancer"
```

Explanation:
- `--servers 1`: One master node.
- `--agents 2`: Two worker nodes.
- `--port "80:80@loadbalancer"`: Maps port 80 from the host to the k3d load balancer (for HTTP ingress).
- `--port "443:443@loadbalancer"`: Maps port 443 for HTTPS ingress.

Check the cluster:
```bash
k3d cluster list
kubectl get nodes
```

---

## **3. Deploy an Ingress Resource**
### **(A) Create a Sample Deployment**
Create a basic `nginx` deployment and service:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply the deployment:

```bash
kubectl apply -f nginx.yaml
```

---

### **(B) Create an Ingress Rule**
Create an `ingress.yaml` file to route traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: "traefik"
spec:
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

Apply the ingress resource:

```bash
kubectl apply -f ingress.yaml
```

---

## **4. Update Hosts File**
Since `myapp.local` is a custom domain, map it to localhost:

```bash
echo "127.0.0.1 myapp.local" | sudo tee -a /etc/hosts
```

---

## **5. Test the Ingress**
Check if the ingress is created:

```bash
kubectl get ingress
```

Now, access `http://myapp.local` in your browser or use:

```bash
curl -v http://myapp.local
```

If everything is set up correctly, it should return an Nginx welcome page.

---

## **Optional: Disable Traefik and Use Nginx Ingress**
If you prefer **Nginx Ingress** instead of Traefik:

1. Create a cluster **without Traefik**:
   ```bash
   k3d cluster create mycluster --servers 1 --agents 2 --port "80:80@loadbalancer" --port "443:443@loadbalancer" --k3s-arg "--disable=traefik@server:0"
   ```

2. Install Nginx Ingress:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
   ```

---

To create a Kubernetes **Service** that selects a pod running an `nginx` image, follow these steps:

---

### **1. Run an Nginx Pod**
First, create a pod using `kubectl run`:

```bash
kubectl run mynginx --image=nginx --port=80 --labels="app=nginx"
```

- `mynginx`: Name of the pod.
- `--image=nginx`: Runs the **nginx** container.
- `--port=80`: Exposes port 80.
- `--labels="app=nginx"`: Adds a label to the pod (important for service selection).

---

### **2. Create a Service for Nginx**
Now, create a **Service** that selects pods with the label `app=nginx`:

```bash
kubectl expose pod mynginx --type=ClusterIP --port=80 --target-port=80 --name=nginx-service
```

- `expose pod mynginx`: Creates a service for the pod.
- `--type=ClusterIP`: Default service type (accessible inside the cluster).
- `--port=80`: The service’s port.
- `--target-port=80`: The pod’s container port.
- `--name=nginx-service`: Name of the service.

---

### **3. Verify the Service**
Check if the service is created:

```bash
kubectl get svc
```

Describe the service to see details:

```bash
kubectl describe svc nginx-service
```

---

### **4. Test the Service**
To test within the cluster, use a temporary busybox pod:

```bash
kubectl run testpod --rm -it --image=busybox -- /bin/sh
```

Then inside the pod, try:

```sh
wget -qO- http://nginx-service
```

If successful, it should return the **Nginx welcome page**.

---

### **Alternative: Create the Service YAML**
If you prefer using a YAML file instead of the `kubectl expose` command:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

Apply it:

```bash
kubectl apply -f nginx-service.yaml
```

---

To **install Rancher using Helm** and set it up with a **custom domain**, follow these steps:  

---

## **1. Install k3d (if not installed)**
If you haven’t installed `k3d`, install it first:

```sh
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

Verify installation:

```sh
k3d version
```

---

## **2. Create a k3d Cluster with Load Balancer and Ingress**
Create a **k3d cluster** that exposes ports `80` and `443`:

```sh
k3d cluster create rancher-cluster \
  --servers 1 --agents 2 \
  --port 80:80@loadbalancer --port 443:443@loadbalancer
```

Verify the cluster:

```sh
kubectl get nodes
```

---

## **3. Install Cert-Manager (Required for Rancher)**
Rancher requires **cert-manager** to handle TLS certificates:

```sh
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Wait for cert-manager pods to be ready:

```sh
kubectl get pods -n cert-manager
```

---

## **4. Add Helm Repository for Rancher**
Add the Rancher Helm repository:

```sh
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update
```

Create a namespace for Rancher:

```sh
kubectl create namespace cattle-system
```

---

## **5. Install Rancher with a Custom Domain**
Replace `rancher.example.com` with your actual domain:

```sh
helm install rancher rancher-latest/rancher --namespace cattle-system --set hostname=rancher-apim.local --set replicas=1 --set bootstrapPassword=admin --set ingress.tls.source=rancher
```

Verify that Rancher is running:

```sh
kubectl get pods -n cattle-system
```

---

## **6. Configure DNS**
You need to map your domain to the k3d load balancer.  

If testing locally, edit your **/etc/hosts** (Linux/macOS) or **C:\Windows\System32\drivers\etc\hosts** (Windows):

```
127.0.0.1 rancher.example.com
```

If using a real domain, create a DNS record pointing to your server’s public IP.

---

## **7. Access Rancher UI**
Now, open your browser and go to:

👉 **https://rancher.example.com**

If you see a certificate warning, bypass it or set up a **Let’s Encrypt TLS certificate**.

---

## **8. (Optional) Configure Let's Encrypt for HTTPS**
To use a valid SSL certificate with **Let’s Encrypt**, modify the Helm installation:

```sh
helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=your-email@example.com
```

This will generate an automatic Let's Encrypt certificate.

---

## **9. Login to Rancher**
- **Username:** `admin`  
- **Password:** `admin` (or the password you set)  

You can now use Rancher to manage your k3d cluster.

---

## **Uninstall Rancher**
If you need to remove Rancher:

```sh
helm uninstall rancher -n cattle-system
kubectl delete namespace cattle-system
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
k3d cluster delete rancher-cluster
```

---

