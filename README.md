# Innovatech EP3 — Orquestación y Automatización en AWS EKS

Proyecto de la Evaluación Parcial N°3 (ISY1101 — Introducción a Herramientas DevOps). Despliegue de la aplicación Innovatech (gestión de despachos y ventas) sobre un clúster de Kubernetes administrado (AWS EKS), con pipeline CI/CD automatizado vía GitHub Actions.

## Arquitectura

```
                         Internet
                            │
                            ▼
                 ┌─────────────────────┐
                 │  AWS Load Balancer   │
                 │   (Service: LB)      │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │     Frontend         │
                 │  (React + Nginx)     │
                 └──────────┬──────────┘
                            │  proxy_pass /api/v1/*
              ┌─────────────┴─────────────┐
              ▼                           ▼
   ┌─────────────────────┐    ┌─────────────────────┐
   │  backend-despachos   │    │   backend-ventas     │
   │  (Spring Boot :8081)│    │  (Spring Boot :8080) │
   └──────────┬──────────┘    └──────────┬──────────┘
              │                          │
              └────────────┬─────────────┘
                            ▼
                 ┌─────────────────────┐
                 │       MySQL          │
                 │   (Deployment :3306) │
                 └─────────────────────┘
```

Todos los servicios corren dentro del clúster EKS `innovatech-cluster`, en el namespace `default`. La comunicación interna (Frontend → Backends → MySQL) usa el DNS de Kubernetes (`backend-despachos`, `backend-ventas`, `mysql`), sin exponer los backends ni la base de datos fuera del clúster.

### Componentes

| Componente | Tecnología | Puerto | Tipo de Service |
|---|---|---|---|
| Frontend | React + Vite, servido por Nginx | 80 | LoadBalancer (público) |
| backend-despachos | Spring Boot 3.4 / Java 21 | 8081 | ClusterIP (interno) |
| backend-ventas | Spring Boot 3.4 / Java 21 | 8080 | ClusterIP (interno) |
| MySQL | MySQL 8.0 | 3306 | ClusterIP (interno) |

### Infraestructura AWS (Terraform)

- **VPC** dedicada con 2 subredes públicas en distintas zonas de disponibilidad (alta disponibilidad).
- **EKS cluster** (`innovatech-cluster`) con un node group de instancias EC2 (`t3.medium`, 2 nodos).
- **3 repositorios ECR**: `innovatech-backend-despachos`, `innovatech-backend-ventas`, `innovatech-frontend`.
- **Roles IAM** reutilizando `LabRole` de AWS Academy Learner Lab (las cuentas de Academy no permiten crear roles IAM personalizados).
- **Security Groups** que permiten tráfico entre nodos y desde el Load Balancer hacia el frontend.

El estado de Terraform (`infra/terraform/terraform.tfstate`) refleja únicamente recursos EKS — no hay infraestructura ECS paralela ni recursos huérfanos de iteraciones anteriores del proyecto.

## Cómo levantar el proyecto

### Requisitos previos

- Cuenta de AWS Academy Learner Lab activa (rol `voclabs`).
- AWS CLI v2, kubectl, Docker (con `buildx`), Terraform >= 1.0, Git.

### 1. Configurar credenciales de AWS

Las credenciales de AWS Academy son temporales (vencen junto con la sesión del laboratorio). Cada vez que se inicie el lab, deben actualizarse:

```bash
aws configure set aws_access_key_id <ACCESS_KEY>
aws configure set aws_secret_access_key <SECRET_KEY>
aws configure set aws_session_token <SESSION_TOKEN>
aws configure set region us-east-1
aws sts get-caller-identity   # verifica que la conexión funciona
```

### 2. Crear la infraestructura (solo la primera vez)

```bash
cd infra/terraform
terraform init
terraform apply
```

Esto crea la VPC, el clúster EKS, el node group y los repositorios ECR. Tarda entre 10 y 15 minutos, principalmente por la creación del control plane de EKS.

### 3. Conectar kubectl al clúster

```bash
aws eks update-kubeconfig --region us-east-1 --name innovatech-cluster
kubectl get nodes   # debe mostrar 2 nodos en estado Ready
```

### 4. Desplegar manualmente (alternativa al pipeline)

```bash
kubectl apply -f infra/k8s/mysql.yml
kubectl rollout status deployment/mysql --timeout=180s

kubectl apply -f infra/k8s/backend-ventas.yml
kubectl apply -f infra/k8s/backend-despachos.yml
kubectl apply -f infra/k8s/frontend.yml
kubectl apply -f infra/k8s/hpa.yml

kubectl get svc frontend   # obtiene la URL pública (EXTERNAL-IP)
```

En condiciones normales no es necesario hacer esto manualmente — el pipeline CI/CD (ver más abajo) se encarga de todo el proceso de build, push y deploy automáticamente.

## Pipeline CI/CD (GitHub Actions)

El archivo `.github/workflows/cd.yml` define un pipeline que se dispara automáticamente con cada `push` a la rama `deploy`:

1. **Checkout** del repositorio.
2. **Build** de las 3 imágenes Docker (`backend-ventas`, `backend-despachos`, `frontend`) con `docker buildx`, etiquetadas con el SHA del commit y `latest`.
3. **Push** de las imágenes a Amazon ECR.
4. **Conexión** al clúster EKS vía `aws eks update-kubeconfig`.
5. **Creación de secrets** de MySQL en Kubernetes a partir de GitHub Secrets (nunca se escriben contraseñas en el código).
6. **Deploy**: aplica los manifiestos de MySQL, espera su `rollout status` (para asegurar que la base de datos acepta conexiones antes de continuar), luego aplica los manifiestos de los backends, frontend y HPA.
7. **Actualización de imágenes** con el tag inmutable del commit (`kubectl set image`).
8. **Verificación de rollout** de cada deployment, con timeout — si algún servicio no queda `Ready`, el pipeline falla explícitamente en vez de reportar éxito falso.
9. **Estado final**: imprime pods, services, HPA y la URL pública del frontend como evidencia del despliegue.

### Secrets requeridos en GitHub (Settings → Secrets and variables → Actions)

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial temporal de AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Credencial temporal de AWS Academy |
| `AWS_SESSION_TOKEN` | Token de sesión temporal de AWS Academy |
| `MYSQL_ROOT_PASSWORD` | Contraseña root de MySQL |
| `MYSQL_DATABASE` | Nombre de la base de datos |
| `MYSQL_USER` | Usuario de aplicación |
| `MYSQL_PASSWORD` | Contraseña del usuario de aplicación |

> Como las credenciales de AWS Academy son temporales, deben actualizarse en GitHub Secrets cada vez que se reinicia el Learner Lab, antes de disparar el pipeline.

### Disparar el pipeline

```bash
git checkout deploy
git merge develop          # trae los últimos cambios
git push origin deploy     # dispara el workflow
```

## Autoscaling (HPA)

Cada uno de los 3 deployments (`backend-despachos`, `backend-ventas`, `frontend`) tiene un `HorizontalPodAutoscaler` configurado en `infra/k8s/hpa.yml`:

- **Métrica**: uso de CPU.
- **Umbral**: 50% de CPU solicitada.
- **Réplicas**: mínimo 1, máximo 4.

Se eligió 50% como umbral intermedio: suficientemente bajo para reaccionar antes de saturar el pod, pero no tan bajo como para escalar innecesariamente ante picos breves de carga.

El HPA depende del **Metrics Server**, que no viene instalado por defecto en EKS y debe agregarse manualmente:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### Verificar el autoscaling

```bash
kubectl get hpa -w
```

En una prueba de carga real (generando peticiones continuas contra `backend-ventas`), el HPA escaló automáticamente de 1 a 4 réplicas en menos de un minuto al superar el umbral de CPU, y redujo las réplicas de vuelta a 1 una vez que la carga cesó y pasó el período de estabilización.

## Problemas encontrados y solución

Esta sección documenta issues reales detectados durante el desarrollo y despliegue, junto con su diagnóstico y resolución — relevante para la defensa técnica (análisis crítico del proceso).

### 1. Reinicio inicial de los backends por orden de arranque

**Síntoma**: los pods de `backend-despachos` y `backend-ventas` mostraban 1 reinicio poco después del primer despliegue.

**Causa**: los backends intentaban conectarse a MySQL antes de que este aceptara conexiones (el contenedor de MySQL estaba `Running`, pero el motor de base de datos aún inicializando). Spring Boot fallaba al primer intento de conexión (`CJCommunicationsException: Communications link failure`) y el pod era reiniciado por Kubernetes; en el segundo intento, MySQL ya estaba listo y la conexión se establecía con normalidad.

**Solución**: se agregaron `readinessProbe` y `livenessProbe` reales en el manifiesto de MySQL (`mysqladmin ping`), y el pipeline CI/CD espera explícitamente `kubectl rollout status deployment/mysql` antes de desplegar los backends — equivalente al `depends_on: condition: service_healthy` de Docker Compose, que Kubernetes no ofrece de forma nativa.

### 2. Metrics Server no instalado por defecto en EKS

**Síntoma**: el HPA mostraba `cpu: <unknown>/50%` y no reaccionaba a la carga.

**Causa**: a diferencia de Minikube o Docker Desktop, EKS no incluye el Metrics Server preinstalado; sin él, el HPA no tiene de dónde leer las métricas de uso de CPU/memoria de los pods.

**Solución**: instalación manual del Metrics Server (ver sección de Autoscaling).

### 3. Falta de almacenamiento persistente para MySQL (pendiente)

**Estado actual**: el volumen de datos de MySQL usa `emptyDir`, que no sobrevive a un reinicio o reprogramación del pod. Se evaluó migrar a un `PersistentVolumeClaim` con `StorageClass: gp2`, pero el aprovisionamiento del volumen EBS requiere el addon **EBS CSI Driver**, que no está instalado en el clúster actual. Queda documentado como mejora pendiente para un entorno productivo real.

### 4. Rama `deploy` desincronizada del código real

**Síntoma**: el pipeline CI/CD no tenía nada sustancial que construir.

**Causa**: el proyecto se desarrolló inicialmente en la rama `develop` (con un historial de Git reescrito para excluir archivos pesados/sensibles), pero nunca se hizo merge hacia la rama `deploy`, que es la que dispara el workflow.

**Solución**: `git merge origin/develop --allow-unrelated-histories` para unificar ambas historias y sincronizar el contenido real del proyecto hacia `deploy`.

## Validación funcional

Comunicación verificada end-to-end dentro del clúster:

```bash
# Desde el pod del frontend, hacia los backends vía DNS interno de Kubernetes
kubectl exec -it deploy/frontend -- curl -s http://backend-ventas:8080/api/v1/ventas
kubectl exec -it deploy/frontend -- curl -s http://backend-despachos:8081/api/v1/despachos
```

Ambos backends responden `[]` (lista vacía válida — la conexión a MySQL, la ejecución de la consulta JPA y la serialización a JSON funcionan correctamente; la base de datos simplemente no tiene registros cargados aún).

El frontend es accesible públicamente a través de la URL expuesta por el Service de tipo `LoadBalancer`:

```bash
kubectl get svc frontend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

## Estructura del repositorio

```
.
├── .github/workflows/cd.yml          # Pipeline CI/CD
├── back-Despachos_SpringBoot/        # Microservicio de despachos
├── back-Ventas_SpringBoot/           # Microservicio de ventas
├── front_despacho/                   # Frontend React
├── infra/
│   ├── k8s/                          # Manifiestos de Kubernetes
│   └── terraform/                    # Infraestructura como código (AWS)
└── docker-compose.yml                # Entorno de desarrollo local
```