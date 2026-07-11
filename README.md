# 🐶 Tienda de Perritos — Despliegue Cloud en AWS EKS

Aplicación web de microservicios (Frontend, Backend y Base de Datos MySQL) desplegada sobre **Amazon Elastic Kubernetes Service (EKS)**, con autoescalado, auto-healing, monitoreo y un pipeline de **CI/CD con GitHub Actions**.

**Integrantes:** Ignacio Pérez, Benjamin Gonzalez
**Docente:** Rafael Andrés Videla Zamora
**Asignatura:** Introducción a Herramientas DevOps — Sección 001D

---

## 📌 Descripción del proyecto

El objetivo de este proyecto es evidenciar la implementación de una solución tecnológica escalable, resiliente y altamente disponible para la aplicación "Tienda de Perritos", utilizando los servicios de Amazon Web Services (AWS) orquestados mediante Kubernetes.

El sistema se compone de tres capas independientes:

- **Frontend**: interfaz de usuario, expuesta públicamente mediante un Load Balancer.
- **Backend**: lógica de negocio y API, expone el endpoint `/api/productos`.
- **Base de datos (MySQL)**: capa de persistencia, aislada en subredes privadas.

---

## 🏗️ Arquitectura

- **VPC** `eks-vpc` — CIDR `10.0.0.0/16`, 2 zonas de disponibilidad.
- **Subredes**: 2 públicas (Load Balancer / recursos expuestos) + 4 privadas (nodos, backend, base de datos).
- **NAT Gateways**: 1 por AZ (2 en total), para salida a internet desde las subredes privadas.
- **VPC Endpoint**: Gateway de Amazon S3.
- **Security Group** `SG-test-eks-cluster` asociado a `eks-vpc`.
- **Clúster EKS** `test-eks`, con acceso Público y Privado, logging completo (api, audit, authenticator, controllerManager, scheduler) y métricas vía CloudWatch.
- **Node Group** `test-eks-nodo`: instancias Spot `t3.large`, escalado 1 (mínimo) / 1 (deseado) / 3 (máximo).
- **Imágenes Docker** almacenadas en **Amazon ECR** (`tienda-frontend`, `tienda-backend`, `tienda-db`).
- **Exposición pública** mediante un Service tipo `LoadBalancer` en el frontend.

### Complementos (add-ons) del clúster

| Complemento | Función |
|---|---|
| CNI de Amazon VPC | Red de pods, obligatorio |
| CoreDNS | DNS interno del clúster |
| kube-proxy | Comunicación entre servicios |
| Agente de supervisión de nodos | Monitoreo de salud de nodos |
| Servidor de métricas | Requerido por HPA y `kubectl top` |
| Observabilidad de Amazon CloudWatch | Logs del control plane |
| Agente de Amazon EKS Pod Identity | Gestión de identidades a nivel de pod |
| DNS externo | Resolución DNS pública |

---

## 📂 Estructura del repositorio

```
tienda-perritos-eks/
├── backend/
│   └── .dockerignore
├── db/
│   └── .dockerignore
├── frontend/
│   └── .dockerignore
├── k8s/
│   ├── namespace.yaml
│   ├── mysql-secret.yaml
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── backend-hpa.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   └── frontend-hpa.yaml
├── .github/
│   └── workflows/
│       └── deploy-eks.yml
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## 🛠️ Herramientas utilizadas

- Docker Desktop
- AWS CLI
- kubectl
- Visual Studio Code
- Git / GitHub
- GitHub Actions
- Amazon EKS, ECR, VPC, CloudWatch (AWS)

---

## ✅ Requisitos previos

- Docker Desktop, AWS CLI y kubectl instalados.
- Laboratorio de AWS Academy activo (las credenciales expiran al pausarlo).
- Repositorios ECR creados previamente: `tienda-frontend`, `tienda-backend`, `tienda-db`.
- ⚠️ El **ACCOUNT_ID cambia en cada sesión** de AWS Academy: siempre debe verificarse antes de construir cualquier URL de ECR o antes de actualizar los YAMLs de `/k8s`.

---

## 💻 Levantar el entorno en local (Docker Compose)

Para desarrollo y pruebas sin depender de AWS, la aplicación completa puede levantarse en el equipo local con un solo comando:

```bash
docker compose up -d
```

Esto levanta el frontend, el backend y la base de datos MySQL en contenedores locales, replicando la arquitectura de tres capas sin necesidad de infraestructura en la nube. La aplicación queda disponible en `http://localhost:8080`.

Para detener el entorno:

```bash
docker compose down
```

---

## ☁️ Despliegue en AWS EKS

### 1. Configurar credenciales

```bash
aws configure
aws sts get-caller-identity
```

### 2. Conectar kubectl al clúster

```bash
aws eks update-kubeconfig --region us-east-1 --name test-eks
kubectl get nodes
```

### 3. Subir imágenes a Amazon ECR

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin TU_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Por cada capa (frontend, backend, db):
docker build -t tienda-<capa> .
docker tag tienda-<capa>:latest TU_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/tienda-<capa>:eks-v1
docker push TU_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/tienda-<capa>:eks-v1
```

### 4. Desplegar los manifiestos en orden

```bash
kubectl apply -f k8s/namespace.yaml

kubectl apply -f k8s/mysql-secret.yaml
kubectl apply -f k8s/mysql-deployment.yaml
kubectl apply -f k8s/mysql-service.yaml

kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/backend-service.yaml

kubectl apply -f k8s/frontend-deployment.yaml
kubectl apply -f k8s/frontend-service.yaml
```

### 5. Obtener la URL pública

```bash
kubectl get svc tienda-frontend -n tienda
```

El campo `EXTERNAL-IP` entrega un registro DNS del Load Balancer (ej. `a1280...elb.us-east-1.amazonaws.com`) que permite acceder a la aplicación desde el navegador.

---

## 📈 Autoescalado y monitoreo (HPA)

Se implementó Horizontal Pod Autoscaler apoyado en el Metrics Server:

- **Backend**: umbral de CPU 70%, escala de 2 a 10 réplicas.
- **Frontend**: umbral de CPU 60%, escala de 2 a 6 réplicas.

```bash
kubectl get hpa -n tienda
```

### Pruebas de resiliencia realizadas

- **Simulación de carga**: bucle de peticiones contra `/api/productos` desde dentro de un pod, observando el escalado en tiempo real con `kubectl get hpa -n tienda -w`.
- **Auto-healing**: eliminación forzada de un pod (`kubectl delete pod`) y verificación de que el ReplicaSet recrea el pod automáticamente.
- **Fallo a nivel de proceso**: `kill 1` dentro del contenedor, verificando el reinicio automático vía `kubectl describe pod`.
- **Auditoría de recursos**: `kubectl top nodes` / `kubectl top pods` y revisión del log group `/aws/eks/test-eks/cluster` en CloudWatch.

---

## 🔁 Pipeline CI/CD (GitHub Actions)

El workflow `.github/workflows/deploy-eks.yml` automatiza el ciclo de entrega ante cada `push` a la rama `main`:

1. Checkout del código.
2. Autenticación en AWS mediante los *Secrets* del repositorio.
3. Login en Amazon ECR.
4. Generación de un tag único por commit (`GITHUB_SHA::7`).
5. Build y push de las 3 imágenes Docker (frontend, backend, db).
6. Conexión a EKS vía `aws eks update-kubeconfig`.
7. Aplicación de manifiestos base y actualización de imágenes con `kubectl set image`.
8. `kubectl rollout status` — despliegue **zero-downtime**, esperando a que los nuevos Pods estén sanos antes de continuar.

### Secrets requeridos en GitHub

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial de acceso AWS |
| `AWS_SECRET_ACCESS_KEY` | Credencial de acceso AWS |
| `AWS_SESSION_TOKEN` | Token de sesión temporal (laboratorio) |
| `AWS_REGION` | Región, ej. `us-east-1` |
| `EKS_CLUSTER_NAME` | Nombre del clúster, ej. `test-eks` |
| `EKS_NAMESPACE` | Namespace de Kubernetes, ej. `tienda` |

> ⚠️ Cada vez que se pausa y reinicia el laboratorio de AWS Academy, deben actualizarse `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` y `AWS_SESSION_TOKEN`.

---

## 🔒 Seguridad

- **Imágenes base minimalistas (Alpine)**: reducen la superficie de ataque y el tamaño final de la imagen.
- **Gestión de credenciales vía Kubernetes Secrets**: las contraseñas de la base de datos se almacenan codificadas en Base64 mediante `mysql-secret.yaml`, evitando texto plano en el repositorio.
- **Secrets de GitHub Actions**: las credenciales de AWS se inyectan dinámicamente durante la ejecución del pipeline, sin exponerse en el código fuente.
- **`.dockerignore` por capa**: cada carpeta (`frontend/`, `backend/`, `db/`) excluye archivos sensibles o innecesarios (`node_modules`, `.env`, etc.) del contexto de build de Docker.
- **Aislamiento de red**: la base de datos y el backend operan en subredes privadas, sin exposición directa a internet; solo el frontend es accesible públicamente a través del Load Balancer.
- **Namespace dedicado (`tienda`)**: aísla lógicamente los recursos de la aplicación respecto al resto del clúster (`kube-system`).
- **Puertos mínimos expuestos**: únicamente el servicio de frontend se publica como `LoadBalancer`; backend y base de datos usan `ClusterIP` (solo accesibles internamente).

---

## 🧭 Orquestación y escalabilidad — ¿por qué EKS y no EC2 manual?

Se optó por **Amazon EKS** en lugar de aprovisionar instancias EC2 manualmente porque Kubernetes entrega de forma nativa:

- **Auto-healing**: los controladores (ReplicaSet/Deployment) detectan y reemplazan automáticamente los Pods que fallan, sin intervención manual.
- **Autoescalado horizontal (HPA)**: ajuste dinámico del número de réplicas según el consumo de CPU, algo que en EC2 requeriría scripts y Auto Scaling Groups configurados manualmente por servicio.
- **Rolling updates sin downtime**: `kubectl set image` + `kubectl rollout status` permiten actualizar versiones reemplazando Pods de a uno, sin interrumpir el servicio.
- **Monitoreo integrado**: integración directa con CloudWatch y el Metrics Server para observabilidad, sin tener que instrumentar cada instancia por separado.
- **Declaratividad**: toda la infraestructura de la aplicación se define en manifiestos YAML versionables, reproducibles y auditables.

En síntesis, EKS traslada la responsabilidad operativa de mantener disponibilidad, escalado y recuperación ante fallos al plano de control de Kubernetes, reduciendo el trabajo manual y el margen de error frente a una solución basada en EC2 puro.

---

## ✅ Validación

- [x] Clúster EKS activo y nodos en estado `Ready`.
- [x] Pods de las 3 capas en estado `Running`.
- [x] Aplicación accesible públicamente vía `EXTERNAL-IP`.
- [x] HPA escalando bajo carga (backend y frontend).
- [x] Auto-healing verificado ante eliminación forzada de Pods.
- [x] Pipeline CI/CD ejecutado exitosamente en GitHub Actions.
- [x] Entorno local levantado con `docker compose up -d`.

---

## 🧹 Limpieza de recursos (AWS Academy)

Al finalizar cada sesión, para evitar consumo innecesario de créditos:

1. Eliminar el Node Group (`test-eks-nodo`).
2. Eliminar el clúster EKS (`test-eks`).
3. Eliminar las etiquetas `kubernetes.io/cluster/test-eks` y `kubernetes.io/role/elb` de las subredes públicas.
4. Eliminar los NAT Gateways (cobran incluso con el clúster detenido).

---

## 📄 Conclusión

Este proyecto demuestra el despliegue exitoso de una arquitectura de microservicios sobre AWS EKS, con red segura (VPC), autoescalado (HPA), auto-healing, monitoreo (CloudWatch) y un pipeline de CI/CD completamente automatizado con GitHub Actions, cumpliendo con los estándares de la industria DevOps: entrega continua, auditable y libre de intervención manual.
