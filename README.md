# Install Defectdojo on k3s


This document provides step-by-step instructions to install K3s (lightweight Kubernetes), Helm package manager, and DefectDojo.

## Installation Steps

### 1. Install K3s

K3s is a lightweight Kubernetes distribution perfect for edge, IoT, and development environments.

```bash
curl -sfL https://get.k3s.io | sh - 
```

After installation, verify that your node is ready (this typically takes about 30 seconds):

```bash
sudo k3s kubectl get node 
```

### 2. Install Helm

Helm is the package manager for Kubernetes that helps you manage Kubernetes applications.

First, add the Helm repository signing key:

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
```

Install required packages:

```bash
sudo apt-get install apt-transport-https --yes
```

Add the Helm stable repository:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
```

Update package list and install Helm:

```bash
sudo apt-get update
sudo apt-get install helm
```

### 3. Configure Kubernetes Access Permissions

Change ownership of the K3s configuration file to allow the current user to use kubectl without sudo:

```bash
sudo chown $USER:$USER /etc/rancher/k3s/k3s.yaml
```

Set up kubectl configuration:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
```

### 4. Configure Helm Repositories

#### Add DefectDojo Repository

```bash
helm repo add defectdojo 'https://raw.githubusercontent.com/DefectDojo/django-DefectDojo/helm-charts'
```

#### Add Bitnami Repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

#### Update Repositories

```bash
helm repo update
```

#### Search for Available Charts

```bash
helm search repo defectdojo
```

### 5. Install DefectDojo from Source

Clone the DefectDojo repository:

```bash
git clone https://github.com/DefectDojo/django-DefectDojo
```

Navigate to the DefectDojo directory:

```bash
cd django-DefectDojo
```

Update Helm dependencies for DefectDojo:

```bash
helm dependency update ./helm/defectdojo
```

### 6. Deploy DefectDojo

Set environment variables for deployment configuration:

```bash
DJANGO_INGRESS_ENABLED=false
DJANGO_INGRESS_ACTIVATE_TLS=false
```

Install DefectDojo using Helm:

```bash
helm install \
  defectdojo \
  ./helm/defectdojo \
  --set django.ingress.enabled=${DJANGO_INGRESS_ENABLED} \
  --set django.ingress.activateTLS=${DJANGO_INGRESS_ACTIVATE_TLS} \
  --set createSecret=true \
  --set createRedisSecret=true \
  --set createPostgresqlSecret=true
```

### 7. Retrieve Admin Credentials

Get the DefectDojo admin password:

```bash
echo "DefectDojo admin password: $(kubectl \
  get secret defectdojo \
  --namespace=default \
  --output jsonpath='{.data.DD_ADMIN_PASSWORD}' \
  | base64 --decode)"
```

Default admin username: `admin`

### 8. Enable External Access

Change the service type to NodePort to allow external access:

```bash
kubectl patch service defectdojo-django -p '{"spec":{"type":"NodePort"}}'
```

### 9. Configure Allowed Hosts

List available ConfigMaps:

```bash
kubectl get configmaps
```

Edit the DefectDojo ConfigMap to add your server IP to allowed hosts:

```bash
kubectl edit configmap defectdojo
```

In the editor, locate and modify the following section:

```yaml
DD_ADMIN_USER: admin
DD_ALLOWED_HOSTS: defectdojo.default.minikube.local,<replace-with-your-server-IP>
```

Save and exit the editor (usually `:wq` in vim).

### 10. Apply Configuration Changes

After modifying the ConfigMap, restart the DefectDojo pod to apply changes:

```bash
kubectl delete pod defectdojo-django
```

Wait for Kubernetes to automatically recreate the pod with the updated configuration:

```bash
kubectl get pods -w
```

## Accessing DefectDojo

After applying all changes and ensuring the pods are running, you can access DefectDojo:

1. **Find the assigned NodePort**:
   ```bash
   kubectl get svc defectdojo-django
   ```
   Access DefectDojo at `http://<your-server-ip>:<assigned-nodeport>`

2. **Port forwarding** (alternative method):
   ```bash
   kubectl port-forward svc/defectdojo-django 8080:80
   ```
   Access DefectDojo at `http://localhost:8080`

Log in with username `admin` and the password retrieved in step 7.
