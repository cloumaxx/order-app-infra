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
  replicaCount: 1
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

1. Inicia Minikube y habilita Ingress:
   ```bash
   minikube start --cpus=4 --memory=6g
   minikube status
   minikube addons enable ingress
   ```

2. Instala el chart:
   ```bash
   helm upgrade --install order-platform charts/order-platform      -n my-tech      --create-namespace      -f charts/order-platform/values.yaml
   ```

3. Verifica recursos:
   ```bash
   kubectl -n my-tech get deploy,po,svc,ing
   ```

4. Prueba salud del backend (desde dentro del clúster):
   ```bash
   kubectl -n my-tech run curl --rm -it --image=curlimages/curl:8.9.1 --restart=Never --      curl -i http://order-backend-svc:8080/actuator/health
   ```

   **Respuesta esperada:**
   ```
   HTTP/1.1 200
   Content-Type: application/vnd.spring-boot.actuator.v3+json

   {"status":"UP","groups":["liveness","readiness"]}
   ```

---

## 🔄 Argo CD

1. Instalar Argo CD (incluye CRDs):
   ```bash
   kubectl create namespace argocd || true
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   kubectl -n argocd get pods
   ```

2. Crear la aplicación:
   ```bash
   kubectl apply -n argocd -f environments/prod/application.yaml
   kubectl -n argocd get applications
   ```

   **Ejemplo de salida:**
   ```
   NAME             SYNC STATUS   HEALTH STATUS
   order-platform   OutOfSync     Healthy
   ```

3. Acceder a la UI:
   ```bash
   kubectl -n argocd port-forward svc/argocd-server 8080:443
   ```

   Abrir en navegador: [http://localhost:8080](http://localhost:8080)

   Usuario: `admin`  
   Password inicial:
   ```bash
   kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
   ```

---

## 🌐 Acceso a la app

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
- **Ver estado de Ingress**:
  ```bash
  minikube addons enable ingress
  ```
- **Ver estado en Argo CD**:
  ```bash
  kubectl -n argocd get applications
  ```
