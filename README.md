# 🏗️ Order App Infra

Infraestructura declarativa (**Helm + Kubernetes**) para la aplicación **Order App**.  
Este repositorio es observado por **Argo CD** siguiendo el modelo **GitOps**.  
Cada vez que Jenkins actualiza `values.yaml` con un nuevo tag de imagen, Argo CD sincroniza el clúster y aplica los cambios automáticamente.

---

## 📂 Estructura

```
charts/
  order-platform/
    Chart.yaml
    values.yaml
    templates/
      order-backend-configmap.yaml
      order-backend-deployment.yaml
      order-backend-hpa.yaml
      order-backend-ingress.yaml
      order-backend-secret.yaml
      order-backend-service.yaml
environments/
  prod/
    application.yaml   # Definición de la aplicación de Argo CD
```

---

## ⚙️ Requisitos

- Kubernetes local con **Minikube** (o un clúster K8s equivalente).
- **kubectl** y **helm** instalados.
- **Argo CD** desplegado en el clúster (`argocd` namespace).
- Namespace de la app: `my-tech`.

---

## 📝 Variables principales (`values.yaml`)

```yaml
backend:
  image:
    repository: docker.io/eduardoalvear/order-app
    tag: "de520f5"    # actualizado automáticamente por Jenkins
  replicas: 1
  resources:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  env:
    DB_HOST: "order-db-postgresql.my-tech.svc.cluster.local"
    DB_NAME: "orders"
ingress:
  enabled: true
  host: "order.local"
  path: /api
```

---

## 🚀 Despliegue manual con Helm  

```bash
kubectl create namespace my-tech || true

helm upgrade --install order-platform charts/order-platform   -n my-tech   -f charts/order-platform/values.yaml
```

---

## 🔄 Argo CD

### `environments/prod/application.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'git@github.com:cloumaxx/order-app-infra.git'
    targetRevision: main
    path: charts/order-platform
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: my-tech
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Crear la aplicación en Argo CD
```bash
kubectl apply -n argocd -f environments/prod/application.yaml
```

---

## 🌐 Acceso

- **Con Ingress habilitado en Minikube**:
  ```bash
  echo "$(minikube ip) order.local" | sudo tee -a /etc/hosts
  curl http://order.local/api/actuator/health
  ```

- **Sin Ingress**:
  ```bash
  kubectl -n my-tech port-forward svc/order-backend-svc 8080:8080
  curl http://localhost:8080/actuator/health
  ```

---

## 🔁 Flujo CI/CD

1. Jenkins construye la imagen → `docker.io/eduardoalvear/order-app:<SHA>`.
2. Jenkins actualiza `values.yaml` con el nuevo tag y hace commit en este repo.
3. Argo CD detecta el cambio y sincroniza el clúster.
4. Kubernetes aplica el rolling update del Deployment.

---

## 🛠️ Troubleshooting

- **Pods no listos**:
  ```bash
  kubectl -n my-tech describe deploy/order-backend
  kubectl -n my-tech logs -l app=order-backend --tail=100
  ```
- **Ingress en Minikube**:
  ```bash
  minikube addons enable ingress
  ```
- **Ver estado en Argo CD**:
  ```bash
  kubectl -n argocd get applications
  ```
