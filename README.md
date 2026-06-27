# EA Parcial 3 — Despliegue de Redmine + PostgreSQL en Kubernetes (AWS EKS)

**Asignatura:** DIY7111 — Tecnologías de Virtualización · DuocUC
**Caso:** Consultora PMOTrack
**Autor:** NOMBRE

---

## 1. Descripción de la arquitectura

Se despliega la herramienta de gestión de proyectos **Redmine** (frontend web) con su base de
datos **PostgreSQL**, sobre un cluster de **Kubernetes en AWS EKS**, dentro del namespace
`pmotrack`. La solución es una aplicación de **dos capas**:

```
                 Internet
                    │
                    ▼
        ┌───────────────────────┐
        │  Service NodePort      │  redmine  (puerto 30080 -> 3000)
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │  Deployment Redmine    │  redmine:5   (HPA: 1 a 5 réplicas)
        └───────────┬───────────┘
                    │  conecta por nombre de Service
                    ▼
        ┌───────────────────────┐
        │  Service ClusterIP     │  postgres (puerto 5432, solo interno)
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │  Deployment PostgreSQL │  postgres:16
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │  PV + PVC (hostPath)   │  /var/lib/postgresql/data
        └───────────────────────┘
```

- **Redmine** se expone al exterior mediante un **Service NodePort** y se accede desde el
  navegador en `IP:PUERTO`.
- **Redmine se conecta a PostgreSQL** usando el nombre del Service (`postgres`) como host,
  gracias al DNS interno de Kubernetes.
- **PostgreSQL** sólo se expone internamente mediante un **Service ClusterIP** (puerto 5432).
- Los datos de PostgreSQL se persisten en un **PersistentVolume / PersistentVolumeClaim**,
  por lo que sobreviven a la eliminación y recreación de los Pods.
- El frontend Redmine escala automáticamente con un **HorizontalPodAutoscaler** (HPA).

## 2. Decisiones técnicas

| Decisión | Elección | Justificación |
|---|---|---|
| Imagen de la base de datos | `postgres:16` | Versión LTS estable, sugerida por el caso. |
| Imagen del frontend | `redmine:5` | Versión sugerida; soporta PostgreSQL por variables de entorno. |
| Credenciales | **Secret** (`postgres-secret`) | Las contraseñas no se escriben en texto plano dentro de los Deployments; se inyectan vía `envFrom`/`secretKeyRef`. |
| Almacenamiento | **PV + PVC** con `hostPath` y `storageClassName: manual` | El Learner Lab no requiere EBS para esta práctica; el `hostPath` vive en el disco del nodo y persiste mientras el cluster exista. En producción se usaría EBS (`gp3`), que persiste de forma independiente al nodo. |
| `PGDATA` en subdirectorio | `/var/lib/postgresql/data/pgdata` | Evita el conflicto del directorio `lost+found` al montar un volumen sobre la raíz de datos. |
| Service de PostgreSQL | **ClusterIP** (5432) | La BD sólo debe ser accesible internamente por Redmine, no desde Internet. |
| Service de Redmine | **NodePort** (30080) | Permite el acceso web externo en el Learner Lab sin un LoadBalancer/Ingress. |
| Autoescalado | **HPA** v2, CPU 50%, mín 1 / máx 5 | Cumple el requisito del caso; requiere `resources.requests.cpu` en el Deployment de Redmine para calcular el % de uso. |

## 3. Instrucciones de despliegue paso a paso

> Todos los comandos se ejecutan desde **AWS CloudShell**.

### Paso 0 — Clonar el repositorio y crear el cluster
```bash
git clone https://github.com/raggon-cl/iny1105-ea3-base.git
cd iny1105-ea3-base
bash commons/scripts/setup-cloudshell.sh   # solo si es una sesión nueva
bash commons/scripts/create-cluster.sh     # crea el cluster EKS (tarda varios minutos)
kubectl get nodes                          # EVIDENCIA 01: nodos en estado Ready
```

### Paso 1 — Crear el namespace y el Secret
```bash
kubectl apply -f manifests/namespace.yaml
kubectl apply -f manifests/postgres-secret.yaml
kubectl get secret -n pmotrack
```

### Paso 2 — Desplegar PostgreSQL con persistencia
```bash
kubectl apply -f manifests/postgres-storage.yaml
kubectl apply -f manifests/postgres-deployment.yaml
kubectl apply -f manifests/postgres-service.yaml

kubectl get pvc -n pmotrack                # EVIDENCIA 03: PVC en estado Bound
kubectl get pods -n pmotrack -w            # esperar a que postgres esté Running
```

### Paso 3 — Desplegar Redmine y exponerlo
```bash
kubectl apply -f manifests/redmine-deployment.yaml
kubectl apply -f manifests/redmine-service.yaml

# Abrir el puerto NodePort en el Security Group
bash commons/scripts/open-nodeport.sh 30080

# Obtener la IP externa del nodo
kubectl get nodes -o wide
```
Abre en el navegador `http://<IP-EXTERNA>:30080` → **EVIDENCIA 04** (Redmine funcionando).

### Paso 4 — Configurar el autoscaling
```bash
# Instalar Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl apply -f manifests/hpa.yaml
kubectl get hpa -n pmotrack                # las métricas deben dejar de aparecer <unknown>

# Generar carga (en otra terminal de CloudShell)
kubectl run -it --rm carga --image=busybox -n pmotrack --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://redmine; done"

kubectl get hpa -n pmotrack -w             # EVIDENCIA 05: réplicas escalando
kubectl get pods -n pmotrack               # se ven varios Pods de redmine
```

### Paso 5 — Verificar la persistencia (en la MISMA sesión)
```bash
# 1. En Redmine, crea un proyecto de prueba desde el navegador.
# 2. Elimina el Pod de PostgreSQL:
kubectl delete pod -l app=postgres -n pmotrack
# 3. Kubernetes lo recrea automáticamente:
kubectl get pods -n pmotrack -w
# 4. Recarga Redmine: el proyecto de prueba SIGUE EXISTIENDO  -> EVIDENCIA 06
```

### Paso 6 — Ver el estado general
```bash
kubectl get all -n pmotrack                # EVIDENCIA 02: todos los objetos
```

### Al terminar cada sesión (IMPORTANTE)
```bash
bash commons/scripts/delete-cluster.sh     # elimina el cluster y libera crédito del Learner Lab
```

## 4. Evidencias

Las capturas se encuentran en la carpeta [`evidencias/`](./evidencias):

| Archivo | Demuestra |
|---|---|
| `01-nodes.png` | Cluster EKS con nodos en estado Ready. |
| `02-get-all.png` | Todos los objetos del namespace `pmotrack`. |
| `03-pvc-bound.png` | PVC de PostgreSQL en estado Bound. |
| `04-redmine-navegador.png` | Redmine funcionando en el navegador. |
| `05-hpa-carga.png` | El HPA escalando réplicas bajo carga. |
| `06-persistencia.png` | El proyecto de Redmine persiste tras recrear el Pod de PostgreSQL. |
