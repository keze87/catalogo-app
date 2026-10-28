# Informe - Trabajo Práctico N° 6

**Asignatura:** Contenedores, Docker y Orquestación con Kubernetes
**Variante:** C - catalogo-app [React + Vite, Python + FastAPI, MongoDB 7]
**Alumno:** César Ezequiel Herrera

---

## 0. Preparación del Entorno y Limpieza Previa

Antes de iniciar con los nuevos controladores, se eliminaron todos los Pods sueltos de la entrega anterior dentro del namespace de la aplicación (conservando sus manifiestos en `k8s/`):

```bash
kubectl delete pod --all -n catalogo-app
```

Verificación del contexto activo y obtención de la IP interna del nodo:

```bash
kubectl config current-context
kubectl get nodes -o wide
```

* **IP del Nodo (`INTERNAL-IP`):** `[Registrar aquí la IP del nodo, ej: 172.16.21.17]`

---

## 1. Manifiestos Declarativos: Services y Deployments

Se diseñaron tres objetos `Service` y tres objetos `Deployment` distribuidos según los requisitos de exposición y escalabilidad:

| Objeto / Capa | Tipo de Service | Réplicas | Justificación Arquitectónica |
| --- | --- | --- | --- |
| `catalogo-db` | `ClusterIP` | 1 | La base de datos no debe exponerse fuera del clúster; su réplica única evita inconsistencias al no disponer de clusterización activa (*replica set*). |
| `catalogo-api` | `ClusterIP` | 2 | Su único cliente es el reverse proxy de Nginx desde el interior de la red del clúster. |
| `catalogo-frontend` | `NodePort` (puerto `30302`) | 2 | Constituye el único punto de entrada expuesto para los clientes web externos. |

### a) Capa de Base de Datos (`catalogo-db`)

#### `k8s/svc-catalogo-db.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalogo-db
  namespace: catalogo-app
spec:
  type: ClusterIP
  selector:
    app: catalogo-db
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

#### `k8s/deploy-catalogo-db.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogo-db
  namespace: catalogo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalogo-db
  template:
    metadata:
      labels:
        app: catalogo-db
    spec:
      containers:
        - name: mongo
          image: mongo:7
          ports:
            - containerPort: 27017
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              value: catalogo_user
            - name: MONGO_INITDB_ROOT_PASSWORD
              value: catalogo_pass
```

### b) Capa de Backend (`catalogo-api`)

#### `k8s/svc-catalogo-api.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalogo-api
  namespace: catalogo-app
spec:
  type: ClusterIP
  selector:
    app: catalogo-api
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
```

#### `k8s/deploy-catalogo-api.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogo-api
  namespace: catalogo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalogo-api
  template:
    metadata:
      labels:
        app: catalogo-api
    spec:
      containers:
        - name: catalogo-api
          image: keze87/catalogo-api:v1
          ports:
            - containerPort: 8000
          env:
            - name: DB_HOST
              value: catalogo-db
            - name: DB_PORT
              value: "27017"
            - name: DB_NAME
              value: catalogodb
            - name: DB_USER
              value: catalogo_user
            - name: DB_PASSWORD
              value: catalogo_pass
            - name: ROOT_PATH
              value: /api
            - name: SEED_DEMO
              value: "true"
```

### c) Capa de Frontend (`catalogo-frontend`)

#### `k8s/svc-catalogo-frontend.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalogo-frontend
  namespace: catalogo-app
spec:
  type: NodePort
  selector:
    app: catalogo-frontend
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
      nodePort: 30302
```

#### `k8s/deploy-catalogo-frontend.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogo-frontend
  namespace: catalogo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalogo-frontend
  template:
    metadata:
      labels:
        app: catalogo-frontend
    spec:
      containers:
        - name: frontend
          image: keze87/catalogo-frontend:v1
          ports:
            - containerPort: 8080
          env:
            - name: API_HOST
              value: catalogo-api
            - name: API_PORT
              value: "8000"
```

---

## 2. Orden de Aplicación y Simulación de Fallo de Upstream

### a) Simulación deliberada de fallo

Se aplicó el `Deployment` de frontend sin que existiera previamente el `Service` de la API (`catalogo-api`) en el clúster:

```bash
kubectl delete svc catalogo-api -n catalogo-app --ignore-not-found
kubectl apply -f k8s/deploy-catalogo-frontend.yaml
```

**Registro de logs del Pod de frontend:**

```text
nginx: [emerg] host not found in upstream "catalogo-api" in /etc/nginx/conf.d/default.conf:24
```

* **Diagnóstico técnico:** Nginx valida la resolución de nombres DNS de sus directivas de upstream durante la inicialización de sus procesos de trabajo. Como el nombre DNS en Kubernetes depende de la existencia de un objeto `Service` registrado en CoreDNS, el Pod cae en bucle de reinicios (*CrashLoopBackOff*) si el `Service` no ha sido dado de alta previamente.

### b) Aplicación en orden correcto

Se aplican los objetos `Service` en primer lugar (creando inmediatamente los registros DNS estables) y posteriormente los `Deployments`:

```bash
kubectl apply -f k8s/svc-catalogo-db.yaml
kubectl apply -f k8s/svc-catalogo-api.yaml
kubectl apply -f k8s/svc-catalogo-frontend.yaml

kubectl apply -f k8s/deploy-catalogo-db.yaml
kubectl apply -f k8s/deploy-catalogo-api.yaml
kubectl apply -f k8s/deploy-catalogo-frontend.yaml
```

---

## 3. Verificación de Selectors y Endpoints

Comando de verificación ejecutado:

```bash
kubectl get endpoints -n catalogo-app
```

**Salida obtenida:**

```text
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME                ENDPOINTS                           AGE
catalogo-api        10.244.0.27:8000,10.244.0.28:8000   7m56s
catalogo-db         10.244.0.26:27017                   7m59s
catalogo-frontend   10.244.0.24:8080,10.244.0.29:8080   7m56s
```

* **Advertencia observada:** `Warning: v1 Endpoints is deprecated in v1.33+, use discovery.k8s.io/v1 EndpointSlice`. Esta advertencia del API Server es puramente informativa debido a la migración paulatina hacia `EndpointSlice`.

* **Análisis de vinculación:** Cada `Service` empareja sus puertos con las direcciones IP internas de los Pods generados por el `Deployment` gracias a la coincidencia estricta entre la clave `spec.selector` del `Service` y la etiqueta `metadata.labels` de la plantilla del Pod. Si los selectores no coincidieran, la columna `ENDPOINTS` mostraría `<none>`.

---

## 4. Acceso al Frontend y Carga de Producto

* **URL de acceso:** `http://172.16.21.17:30302`

* Se procedió a ingresar a la interfaz de usuario en el navegador y dar de alta un producto nuevo con imagen adjunta.

**Confirmación del producto creado:**

```bash
curl -s http://172.16.21.17:30302/api/productos | grep -o '"nombre":"[^"]*"' | grep ejemplo
```

* **Resultado:**

```json 
"nombre":"Sorny ejemplo"
```

---

## 5. Service NodePort de Depuración para la API

### Manifiesto `k8s/svc-catalogo-api-nodeport.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: catalogo-api-nodeport
  namespace: catalogo-app
spec:
  type: NodePort
  selector:
    app: catalogo-api
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
      nodePort: 30502
```

Aplicación y prueba de disponibilidad:

```bash
kubectl apply -f k8s/svc-catalogo-api-nodeport.yaml
curl -s http://172.16.21.17:30502/health
```

**Salida obtenida:**

```json
{"status":"ok","db":"ok","version":"v1"}
```

> **Aclaración de seguridad y diseño:** El objeto `catalogo-api-nodeport` se implementa de forma provisoria con fines exclusivos de depuración en este práctico. En el despliegue final con Helm (Semana 8), el backend operará de forma estrictamente privada mediante `ClusterIP`.
>
>

---

## 6. Comprobación de Autorreparación (*Self-Healing*)

Con la aplicación en ejecución continua, se eliminó un Pod específico del backend:

```bash
POD_API=$(kubectl get pod -l app=catalogo-api -n catalogo-app -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD_API -n catalogo-app
kubectl get pods -l app=catalogo-api -n catalogo-app -w
```

**Salida del seguimiento:**

```text
NAME                            READY   STATUS    RESTARTS      AGE
catalogo-api-5dd687c4f7-ctf5p   1/1     Running   1 (32m ago)   35m
catalogo-api-5dd687c4f7-z8v6c   1/1     Running   1 (32m ago)   35m
catalogo-api-5dd687c4f7-ctf5p   1/1     Terminating   1 (34m ago)   37m
catalogo-api-5dd687c4f7-hmq2j   0/1     Pending       0             1s
catalogo-api-5dd687c4f7-ctf5p   1/1     Terminating   1 (34m ago)   37m
catalogo-api-5dd687c4f7-hmq2j   0/1     Pending       0             7s
catalogo-api-5dd687c4f7-hmq2j   0/1     ContainerCreating   0             12s
catalogo-api-5dd687c4f7-ctf5p   0/1     Completed           1 (35m ago)   37m
catalogo-api-5dd687c4f7-ctf5p   0/1     Completed           1 (35m ago)   37m
catalogo-api-5dd687c4f7-ctf5p   0/1     Completed           1 (35m ago)   38m
catalogo-api-5dd687c4f7-hmq2j   0/1     ContainerCreating   0             35s
catalogo-api-5dd687c4f7-hmq2j   1/1     Running             0             37s
```

* **Contraste con la Semana 5:** En el TP5, un Pod desnudo (*naked pod*) eliminado desaparecía de forma definitiva del clúster. En el TP6, el controlador `Deployment` (a través de su `ReplicaSet`) mantiene el bucle de reconciliación declarativo: detecta que la cantidad de réplicas activas cayó por debajo de `replicas: 2` e instancia inmediatamente un Pod nuevo de reemplazo, sin interrupción perceptible del servicio.

---

## 7. Actualización Progresiva (*Rolling Update*) y Rollback

### a) Compilación y publicación de la versión v2

```bash
docker build --build-arg APP_VERSION=v2 -t keze87/catalogo-api:v2 ./backend
docker push keze87/catalogo-api:v2
```

### b) Ejecución del Rolling Update

```bash
kubectl set image deployment/catalogo-api catalogo-api=keze87/catalogo-api:v2 -n catalogo-app
kubectl rollout status deployment/catalogo-api -n catalogo-app
```

**Salida de `rollout status`:**

```text
deployment "catalogo-api" successfully rolled out
```

* **Verificación de versión en el frontend / API:** El endpoint `http://172.16.21.17:30502/health` y el indicador del frontend informan:

```json
{"status":"ok","db":"ok","version":"v2"}
```

### c) *Rollback*

Inspección del historial y marcha atrás de la actualización:

```bash
kubectl rollout history deployment/catalogo-api -n catalogo-app
kubectl rollout undo deployment/catalogo-api -n catalogo-app
kubectl rollout status deployment/catalogo-api -n catalogo-app
```

**Salida de `rollout history` y verificación post-undo:**

```text
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
```

```json
{"status":"ok","db":"ok","version":"v1"}
```

* **Resultado:** El deployment revirtió ordenadamente sus réplicas a la versión `v1` sin caídas de disponibilidad.

---

## 8. Réplicas sin Estado y Almacenamiento en Memoria/Base de Datos

### a) Escalado a 3 réplicas del backend

```bash
kubectl scale deployment catalogo-api --replicas=3 -n catalogo-app

for i in $(seq 1 10); do
  curl -s -o /dev/null -w "%{http_code} " http://172.16.21.17:30502/productos
done
echo ""
```

**Resultado:** Se obtienen respuestas `200` sucesivas distribuidas mediante el `Service` entre las tres réplicas activas.

### b) Comprobación de aislamiento en el sistema de archivos de los Pods

Se escribió un archivo de prueba en el disco de una réplica específica y se buscó su presencia en otra:

```bash
# Escribir en la primera réplica
kubectl exec deploy/catalogo-api -n catalogo-app -c catalogo-api -- sh -c 'echo prueba > /tmp/foto.bin; ls -l /tmp/foto.bin'

# Obtener nombre de otro pod de la API
OTRO_POD=$(kubectl get pods -l app=catalogo-api -n catalogo-app -o jsonpath='{.items[2].metadata.name}')

# Consultar el archivo en la segunda réplica
kubectl exec $OTRO_POD -n catalogo-app -c catalogo-api -- ls -l /tmp/foto.bin
```

**Salida obtenida:**

```text
ls: cannot access '/tmp/foto.bin': No such file or directory
command terminated with exit code 2
```

### c) Pregunta teórica de dominio: ¿Por qué la API debe ser sin estado (*stateless*)?

Cada réplica de un Pod corre en su propio namespace de montaje con un sistema de archivos efímero y totalmente aislado del resto. Si la API almacenara las imágenes subidas en su disco local, una petición atendida por el Pod A persistiría la imagen localmente, pero cualquier solicitud subsecuente enrutada por el `Service` hacia el Pod B o Pod C resultaría en un error HTTP 404 porque el archivo no existe en esos discos. Además, ante el reemplazo o reinicio de un Pod, los datos escritos en su disco local se destruyen.

Por estas razones, la API almacena los bytes de las imágenes dentro de documentos de MongoDB, permitiendo que cualquier réplica acceda de manera consistente a la información.

---

## 9. Limitación Declarada: Falta de Persistencia en la Base de Datos

### a) Eliminación del Pod de MongoDB

```bash
kubectl delete pod -l app=catalogo-db -n catalogo-app
kubectl get pods -l app=catalogo-db -n catalogo-app -w
```

### b) Verificación del catálogo tras recreación del Pod de la BD

Se consultó el catálogo antes de reiniciar la API:

```bash
curl -s http://172.16.21.17:30502/productos
```

* **Resultado:** Devuelve `[]` (catálogo vacío). El producto cargado a mano y los datos sembrados se perdieron debido a que el Pod de MongoDB se ejecutó sin volumen persistente montado.

### c) Reinicio de la API y siembra automática

```bash
kubectl rollout restart deployment/catalogo-api -n catalogo-app
kubectl rollout status deployment/catalogo-api -n catalogo-app
curl -s http://172.16.21.17:30502/productos | grep -o '"id"' | wc -l
```

* **Resultado:** Vuelven a aparecer los 12 productos de demostración porque la API ejecuta la rutina de siembra al iniciar, pero el producto cargado manualmente en el Punto 4 **no se recupera**.

* **Conclusión de la limitación:** `catalogo-db` se encuentra gestionado como un `Deployment` efímero. Esta limitación conocida será subsanada en la Semana 7 mediante la incorporación de `PersistentVolume` y `PersistentVolumeClaim`.
