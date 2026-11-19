# Trabajo Práctico N° 8: Autoescalado Horizontal y Empaquetado con Helm

**Asignatura:** Contenedores, Docker y Orquestación con Kubernetes
**Variante:** C - `catalogo-app` (React + Vite, Python + FastAPI, MongoDB 7)
**Alumno:** César Ezequiel Herrera

---

## 1. Declaración de Recursos (`resources.requests`) y Autoescalador (`catalogo-api-hpa`)

### a) Configuración de recursos en `catalogo-api`

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "300m"
    memory: "256Mi"
```

### b) Definición de `catalogo-api-hpa`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: catalogo-api-hpa
  namespace: catalogo-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: catalogo-api
  minReplicas: 2
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

### c) Preguntas de diseño del autoescalado

* **¿Contra qué número calcula el autoescalador su porcentaje?**
El `HorizontalPodAutoscaler` calcula la utilización porcentual dividiendo el consumo actual de CPU reportado por `metrics-server` contra el valor definido en **`resources.requests.cpu`** (en este caso, `100m`). Si `resources.requests` no está declarado, Kubernetes carece de denominador y la columna `TARGETS` permanece indefinidamente en `<unknown>`.
* **¿Por qué el proyecto tiene un solo autoescalador entre doce objetos?**

1. **`catalogo-db` (MongoDB):** No puede escalarse horizontalmente con un `Deployment` convencional porque cada réplica levantaría una instancia independiente con su propio volumen/directorio de datos sin replicación configurada (*replica set*), rompiendo la consistencia de la información.
2. **`catalogo-frontend` (Nginx):** Sirve archivos estáticos compilados y actúa como proxy reverso ligero; el cuello de botella computacional y de serialización de datos (lectura y entrega de bytes de imágenes desde MongoDB) recae sobre la capa de API, que es completamente *stateless*.

---

## 2. Demostración de Autoescalado bajo Carga Real

### a) Generación de carga contra `/productos/{id}/imagen`

Se lanzaron Pods temporales con `busybox` realizando peticiones continuas al endpoint de imágenes para exigir lectura desde MongoDB y procesamiento en FastAPI:

```bash
PROD_ID=$(curl -s http://catalogo.local/api/productos | grep -o '"id":"[^"]*"' | head -n 1 | cut -d'"' -f4)

for i in 1 2 3; do
  kubectl run carga-$i -n catalogo-app --image=busybox --restart=Never -- \
    /bin/sh -c "while true; do wget -q -O- http://catalogo-api:8000/productos/$PROD_ID/imagen > /dev/null; done"
done
```

### b) Consumo de recursos observado (`kubectl top pods`)

```text
NAME                                 CPU(cores)   MEMORY(bytes)
carga-1                              197m         4Mi
carga-2                              199m         1Mi
carga-3                              195m         1Mi
catalogo-api-ccbb4df49-5g48w         274m         42Mi
catalogo-api-ccbb4df49-664fn         241m         45Mi
catalogo-api-ccbb4df49-w54qz         252m         43Mi
catalogo-db-7956b7cff6-q7njz         150m         139Mi
catalogo-frontend-664966f46b-flfq8   0m           3Mi
catalogo-frontend-664966f46b-j9tlx   0m           3Mi
```

### c) Evolución del HPA: escalado (2 → 6) y descenso tras ventana de estabilización

```bash
kubectl get hpa -n catalogo-app -w
```

```text
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%   2         6         2          84s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 28%/50%   2         6         2          2m15s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 145%/50%   2         6         2          2m30s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 278%/50%   2         6         4          2m45s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 186%/50%   2         6         6          3m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 123%/50%   2         6         6          3m15s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 120%/50%   2         6         6          3m30s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 121%/50%   2         6         6          4m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 120%/50%   2         6         6          4m15s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 121%/50%   2         6         6          4m30s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 29%/50%    2         6         6          4m45s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         6          5m
catalogo-api-hpa   Deployment/catalogo-api   cpu: 1%/50%     2         6         6          8m30s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         6          8m45s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         6          9m31s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         4          9m46s
catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%     2         6         2          10m
```

---

## 3. Empaquetado en Chart de Helm (`catalogo-app/`)

### a) Inventario exacto de los 12 objetos del Chart

| # | Template / Archivo | Tipo (`Kind`) | Nombre del Objeto | Función |
| --- | --- | --- | --- | --- |
| 1 | `cm-catalogo-api.yaml` | `ConfigMap` | `catalogo-api-config` | Parámetros de conexión y ejecución del backend (`DB_HOST`, `DB_PORT`, `DB_NAME`, `ROOT_PATH`, `SEED_DEMO`). |
| 2 | `cm-catalogo-frontend.yaml` | `ConfigMap` | `catalogo-frontend-config` | Host y puerto interno de la API consumidos por Nginx (`API_HOST`, `API_PORT`). |
| 3 | `secret-catalogo-db.yaml` | `Secret` | `catalogo-db-secret` | Credenciales de administrador de MongoDB (`MONGO_INITDB_ROOT_USERNAME` y `PASSWORD`). |
| 4 | `pvc-catalogo-db.yaml` | `PersistentVolumeClaim` | `catalogo-db-pvc` | Solicitud de almacenamiento persistente de `1Gi` (`ReadWriteOnce`) para `/data/db`. |
| 5 | `svc-catalogo-db.yaml` | `Service` (`ClusterIP`) | `catalogo-db` | Resolución DNS interna estable en el puerto `27017` hacia MongoDB. |
| 6 | `svc-catalogo-api.yaml` | `Service` (`ClusterIP`) | `catalogo-api` | Balanceo interno en el puerto `8000` entre las réplicas activas de la API. |
| 7 | `svc-catalogo-frontend.yaml` | `Service` (`NodePort`) | `catalogo-frontend` | Exposición del frontend en el puerto fijo `30302` y backend del Ingress. |
| 8 | `deploy-catalogo-db.yaml` | `Deployment` | `catalogo-db` | Ejecuta la réplica única de MongoDB 7 montando el PVC y el Secret. |
| 9 | `deploy-catalogo-api.yaml` | `Deployment` | `catalogo-api` | Ejecuta las réplicas sin estado de FastAPI con `resources.requests` definidos. |
| 10 | `deploy-catalogo-frontend.yaml` | `Deployment` | `catalogo-frontend` | Ejecuta las réplicas de Nginx sirviendo la SPA y haciendo proxy de `/api`. |
| 11 | `ingress-catalogo-app.yaml` | `Ingress` | `catalogo-app-ingress` | Enruta el tráfico HTTP del host `catalogo.local` hacia `catalogo-frontend:8080`. |
| 12 | `hpa-catalogo-api.yaml` | `HorizontalPodAutoscaler` | `catalogo-api-hpa` | Escala automáticamente `catalogo-api` entre 2 y 6 réplicas según utilización de CPU. |

*(Nota: El Service auxiliar `catalogo-api-nodeport` de la sexta entrega quedó excluido por tratarse exclusivamente de un recurso de depuración).*

---

## 4. Parametrización Efectiva y Decisión de Cadena de Conexión

### a) Parámetros configurables en `values.yaml`

Se parametrizaron de forma efectiva (verificada mediante `helm template --set`):

* Repositorios, tags y políticas de descarga de las imágenes (`frontend.image.tag`, `api.image.tag`, `db.image.tag`).
* Cantidad de réplicas iniciales por capa (`frontend.replicas`, `api.replicas`, `db.replicas`).
* Recursos (`api.resources.requests` / `limits`) y umbrales del autoescalador (`api.hpa.minReplicas`, `maxReplicas`, `targetCPUUtilizationPercentage`).
* Parámetros de red, almacenamiento y host de Ingress (`frontend.service.nodePort`, `db.persistence.size`, `ingress.host`).

### b) Verificación de renderizado previo a la instalación

```bash
# Cantidad de objetos renderizados
helm template catalogo-app ./catalogo-app | grep -c "^kind:"
# 12

# Comprobación de parametrización efectiva (no decorativa)
helm template catalogo-app ./catalogo-app --set api.image.tag=v2 | grep "image: keze87/catalogo-api"
# image: "keze87/catalogo-api:v2"
```

### c) Decisión sobre la cadena de conexión a MongoDB

* **Alternativa elegida:** Entrega de **piezas por separado** (`DB_HOST`, `DB_PORT`, `DB_NAME` desde `catalogo-api-config` y `DB_USER`, `DB_PASSWORD` desde `catalogo-db-secret`).
* **Justificación:** Evita duplicar las credenciales en dos objetos distintos (reutilizando `catalogo-db-secret` tanto para la inicialización de MongoDB como para la API mediante `secretKeyRef`) y delega en el código de FastAPI el ensamblado de la URI con `?authSource=admin`, evitando errores de autenticación por omisión manual en un string monolítico.

---

## 5. Ciclo Completo de Release con Helm

### a) Instalación inicial (`helm install`)

```bash
helm install catalogo-app ./catalogo-app -n catalogo-app --create-namespace
kubectl get all,cm,secret,pvc,ingress,hpa -n catalogo-app
```

```text
NAME                                     READY   STATUS    RESTARTS   AGE
pod/catalogo-api-779fd8c999-4czgn        1/1     Running   0          2m9s
pod/catalogo-api-779fd8c999-rdpnv        1/1     Running   0          2m9s
pod/catalogo-db-7956b7cff6-6d77w         1/1     Running   0          2m9s
pod/catalogo-frontend-664966f46b-2znjq   1/1     Running   0          2m9s
pod/catalogo-frontend-664966f46b-6t8rs   1/1     Running   0          2m9s

NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/catalogo-api        ClusterIP   10.98.95.106    <none>        8000/TCP         2m10s
service/catalogo-db         ClusterIP   10.96.191.122   <none>        27017/TCP        2m10s
service/catalogo-frontend   NodePort    10.103.163.10   <none>        8080:30302/TCP   2m10s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/catalogo-api        2/2     2            2           2m9s
deployment.apps/catalogo-db         1/1     1            1           2m9s
deployment.apps/catalogo-frontend   2/2     2            2           2m9s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/catalogo-api-779fd8c999        2         2         2       2m9s
replicaset.apps/catalogo-db-7956b7cff6         1         1         1       2m9s
replicaset.apps/catalogo-frontend-664966f46b   2         2         2       2m9s

NAME                                                   REFERENCE                 TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/catalogo-api-hpa   Deployment/catalogo-api   cpu: 2%/50%   2         6         2          2m9s

NAME                                 DATA   AGE
configmap/catalogo-api-config        5      2m10s
configmap/catalogo-frontend-config   2      2m10s
configmap/kube-root-ca.crt           1      2m27s

NAME                                        TYPE                 DATA   AGE
secret/catalogo-db-secret                   Opaque               2      2m10s
secret/sh.helm.release.v1.catalogo-app.v1   helm.sh/release.v1   1      2m10s

NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/catalogo-db-pvc   Bound    pvc-965e1c90-5df0-4949-9b79-46a900dc1b07   1Gi        RWO            local-path     <unset>                 2m10s

NAME                                             CLASS   HOSTS            ADDRESS         PORTS   AGE
ingress.networking.k8s.io/catalogo-app-ingress   nginx   catalogo.local   172.16.21.212   80      2m9s
```

### b) Actualización (`helm upgrade` a `v2`), historial y reversión (`helm rollback`)

```bash
# Actualización de la API a v2
helm upgrade catalogo-app ./catalogo-app -n catalogo-app --set api.image.tag=v2
kubectl rollout status deployment/catalogo-api -n catalogo-app

# Consulta de historial
helm history catalogo-app -n catalogo-app
```

```text
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Wed Nov 18 22:13:42 2026        superseded      catalogo-app-0.1.0      v1              Install complete
2               Wed Nov 18 22:22:35 2026        deployed        catalogo-app-0.1.0      v1              Upgrade complete
```

```bash
# Reversión a la revisión 1
helm rollback catalogo-app 1 -n catalogo-app
kubectl rollout status deployment/catalogo-api -n catalogo-app
```

```text
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Wed Nov 18 22:13:42 2026        superseded      catalogo-app-0.1.0      v1              Install complete
2               Wed Nov 18 22:22:35 2026        superseded      catalogo-app-0.1.0      v1              Upgrade complete
3               Wed Nov 18 22:23:46 2026        deployed        catalogo-app-0.1.0      v1              Rollback to 1
```

### c) Evidencia visual e independencia de ciclos de vida

* **Captura antes del upgrade (API en `v1`):** `{"status":"ok","db":"ok","version":"v1"}`
* **Captura después del upgrade (API en `v2`):** `{"status":"ok","db":"ok","version":"v2"}`
* **Captura después del rollback (API nuevamente en `v1`):** `{"status":"ok","db":"ok","version":"v1"}`
* **Observación arquitectónica:** Modificar `api.image.tag` actualiza únicamente los Pods de `catalogo-api` sin afectar ni reiniciar los Pods de `catalogo-frontend`, demostrando que ambos microservicios mantienen imágenes y ciclos de vida completamente independientes.

---

## 6. Pregunta de Arquitectura de la Variante C: Transición de 1 a 3 Réplicas de la API

**¿Qué deja de estar garantizado al pasar de una réplica de la API a tres?**

Al pasar de una única instancia a múltiples réplicas balanceadas por un `Service`, **deja de estar garantizada la afinidad con el estado local del proceso y del sistema de archivos del contenedor**:

1. **Aislamiento del sistema de archivos efímero:** Cada Pod posee su propio namespace de montaje aislado. Si la API guardara las imágenes cargadas en su disco local (`/tmp` o un directorio interno), solo la réplica que recibió el `POST` tendría el archivo; las peticiones `GET /productos/{id}/imagen` posteriores enrutadas por el `Service` hacia las otras dos réplicas fallarían con error `404 Not Found`.
2. **Estado en memoria RAM:** Cualquier caché local, sesión o variable mutable almacenada en memoria del proceso Python deja de ser compartida entre instancias, generando respuestas inconsistentes según qué Pod atienda cada solicitud.

Por este motivo, el diseño de la Variante C exige que la API sea estrictamente **sin estado (*stateless*)**, delegando toda la persistencia —incluidos los bytes binarios de las imágenes— a MongoDB respaldado por su `PersistentVolumeClaim`.
