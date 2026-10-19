# Informe - Trabajo Práctico N° 5

**Asignatura:** Contenedores, Docker y Orquestación con Kubernetes
**Variante:** C - catalogo-app [React + Vite, Python + FastAPI, MongoDB 7]
**Alumno:** César Ezequiel Herrera

---

## 1. Acceso a Imágenes Propias y Simulación de Fallo de Descarga (`ImagePullBackOff`)

### Publicación previa de imágenes en Docker Hub

* Imagen API: `docker.io/keze87/catalogo-api:v1`
* Imagen Frontend: `docker.io/keze87/catalogo-frontend:v1`

### Simulación de descarga con imagen sin usuario del registry

Comando ejecutado para provocar el error deliberado:

```bash
kubectl run test-fail-image --image=catalogo-api:v1 --restart=Never
```

#### Salida de `kubectl describe pod test-fail-image` (Estado y Eventos)

```text
Containers:
  test-fail-image:
    Container ID:
    Image:          catalogo-api:v1
    Image ID:
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-fcxpj (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-fcxpj:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age               From               Message
  ----     ------     ----              ----               -------
  Normal   Scheduled  19s               default-scheduler  Successfully assigned default/test-fail-image to cherrera
  Normal   BackOff    15s               kubelet            spec.containers{test-fail-image}: Back-off pulling image "catalogo-api:v1"
  Warning  Failed     15s               kubelet            spec.containers{test-fail-image}: Error: ImagePullBackOff
  Normal   Pulling    4s (x2 over 16s)  kubelet            spec.containers{test-fail-image}: Pulling image "catalogo-api:v1"
  Warning  Failed     3s (x2 over 15s)  kubelet            spec.containers{test-fail-image}: Failed to pull image "catalogo-api:v1": failed to pull and unpack image "docker.io/library/catalogo-api:v1": failed to resolve reference "docker.io/library/catalogo-api:v1": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     3s (x2 over 15s)  kubelet            spec.containers{test-fail-image}: Error: ErrImagePull
  ```

* Nombre de imagen que intentó descargar el kubelet: `docker.io/library/catalogo-api:v1`

* Explicación técnica: Al omitir el prefijo de usuario en la definición de la imagen, el runtime de contenedores del nodo asume por omisión el namespace oficial (`library/`) de Docker Hub. Como la imagen reside en un repositorio particular y no en la biblioteca pública oficial de Docker, el registry retorna un error HTTP 404 (Not Found), derivando en el estado `ImagePullBackOff`.

---

## 2. Configuración de Contexto y Namespace `catalogo-app`

### a) Contexto activo (`kubectl config current-context`)

```text
cherrera
```

### b) Creación y fijación del namespace por defecto

```bash
kubectl create namespace catalogo-app
kubectl config set-context --current --namespace=catalogo-app
```

**Verificación:**

```bash
kubectl config view --minify | grep namespace:
```

**Salida:**

```text
namespace: catalogo-app
```

---

## 3. Manifiestos de Pods (`k8s/`)

### a) Generación e incorporación de variables en `k8s/pod-catalogo-db.yaml`

Comando utilizado para generar la plantilla base:

```bash
kubectl run catalogo-db --image=mongo:7 --port=27017 --dry-run=client -o yaml > k8s/pod-catalogo-db.yaml
```

Contenido final de `k8s/pod-catalogo-db.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: catalogo-db
  name: catalogo-db
spec:
  containers:
  - image: mongo:7
    name: catalogo-db
    ports:
    - containerPort: 27017
    env:
      - name: MONGO_INITDB_ROOT_USERNAME
        value: catalogo_user
      - name: MONGO_INITDB_ROOT_PASSWORD
        value: catalogo_pass
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

### b) Manifiesto de la API (`k8s/pod-catalogo-api.yaml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: catalogo-api
  name: catalogo-api
spec:
  containers:
  - image: keze87/catalogo-api:v1
    name: catalogo-api
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
      - name: APP_VERSION
        value: v1
      - name: ROOT_PATH
        value: /api
      - name: SEED_DEMO
        value: "true"
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

### c) Manifiesto del Frontend (`k8s/pod-catalogo-frontend.yaml`)

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: catalogo-api
  name: catalogo-api
spec:
  containers:
  - image: keze87/catalogo-frontend:v1
    name: catalogo-frontend
    ports:
    - containerPort: 8080
    env:
      - name: API_HOST
        value: catalogo-api
      - name: API_PORT
        value: "8000"
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

---

## 4. Despliegue Inicial y Asignación de IPs de Pods

Comando ejecutado para aplicar los dos manifiestos:

```bash
kubectl apply -f k8s/pod-catalogo-db.yaml
kubectl apply -f k8s/pod-catalogo-api.yaml
```

### Salida de `kubectl get pods -o wide`

```text
NAME                READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
catalogo-api        1/1     Running   0          44s   10.244.0.16   cherrera   <none>           <none>
catalogo-db         1/1     Running   0          42m   10.244.0.13   cherrera   <none>           <none>
```

### Registro de direcciones IP asignadas

* **`catalogo-db`:** 10.244.0.13

* **`catalogo-api`:** 10.244.0.16

---

## 5. Diagnóstico del Fallo de la API (`CrashLoopBackOff`)

### a) Salida de `kubectl describe pod catalogo-api`

```text
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-gfpf8:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason   Age                    From     Message
  ----     ------   ----                   ----     -------
  Normal   Pulled   4m51s (x160 over 15h)  kubelet  spec.containers{catalogo-api}: Container image "keze87/catalogo-api:v1" already present on machine and can be accessed by the pod
  Normal   Created  4m49s (x160 over 15h)  kubelet  spec.containers{catalogo-api}: Container created
  Warning  BackOff  66s (x689 over 15h)    kubelet  spec.containers{catalogo-api}: Back-off restarting failed container catalogo-api in pod catalogo-api_catalogo-app(8484517f-83a7-4424-aeb9-31cc46016e7f)
```

### b) Salida de `kubectl logs catalogo-api --previous`

```text
  File "/home/appuser/.local/lib/python3.12/site-packages/pymongo/topology.py", line 303, in _select_servers_loop
    raise ServerSelectionTimeoutError(
pymongo.errors.ServerSelectionTimeoutError: catalogo-db:27017: [Errno -2] Name or service not known (configured timeouts: socketTimeoutMS: 20000.0ms, connectTimeoutMS: 20000.0ms), Timeout: 3.0s, Topology Description: <TopologyDescription id: 6ad65f231c5c35c86ff23f5d, topology_type: Unknown, servers: [<ServerDescription ('catalogo-db', 27017) server_type: Unknown, rtt: None, error=AutoReconnect('catalogo-db:27017: [Errno -2] Name or service not known (configured timeouts: socketTimeoutMS: 20000.0ms, connectTimeoutMS: 20000.0ms)')>]>

ERROR:    Application startup failed. Exiting.
```

### c) Análisis del error registrado (`ServerSelectionTimeoutError` vs `Authentication failed`)

* Mensaje textual obtenido: `pymongo.errors.ServerSelectionTimeoutError: catalogo-db:27017: [Errno -2] Name or service not known`

* Tipo de error: Corresponde a `ServerSelectionTimeoutError` y **no** a `Authentication failed`.

* Justificación técnica: Un Pod por sí solo no registra ningún nombre resoluble en el DNS interno del clúster (CoreDNS). Como todavía no existe un objeto `Service` con el nombre `catalogo-db`, la resolución de host falla a nivel de red. El driver de PyMongo nunca llega a comunicarse con el motor de base de datos ni a enviar las credenciales, de modo que el error jamás podría ser un rechazo de autenticación.

---

## 6. Conexión por Dirección IP y su Carácter Efímero

### a) Configuración temporal con IP de Pod y estado Running

Se modificó `k8s/pod-catalogo-api.yaml` reemplazando `DB_HOST: catalogo-db` por la IP asignada a `catalogo-db`.

Salida de `kubectl get pods` tras reaplicar la API:

```text
NAME                READY   STATUS    RESTARTS   AGE
catalogo-api        1/1     Running   0          3m58s
catalogo-db         1/1     Running   0          17h
```

### b) Eliminación y recreación de `catalogo-db`

```bash
kubectl delete pod catalogo-db
kubectl apply -f k8s/pod-catalogo-db.yaml
kubectl get pod catalogo-db -o wide
```

* IP anterior de `catalogo-db`: 10.244.0.13

* Nueva IP asignada a `catalogo-db`: 10.244.0.19

Salida de `kubectl get pods` (Estado de `catalogo-api` tras el cambio):

```text
NAME                READY   STATUS    RESTARTS        AGE     IP            NODE       NOMINATED NODE   READINESS GATES
catalogo-api        0/1     Error     4 (2m10s ago)   7m11s   10.244.0.20   cherrera   <none>           <none>
catalogo-db         1/1     Running   0               12m     10.244.0.19   cherrera   <none>           <none>
```

### c) Conclusión sobre las IPs de Pods

La dirección IP de un Pod es efímera, volátil y descartable. Al morir o recrearse un Pod, el clúster le asigna una IP diferente tomada del rango de la red de pods, rompiendo cualquier enlace configurado de forma estática. Este comportamiento demuestra la necesidad indispensable de los objetos `Service`, los cuales proveen un punto de acceso estable con IP virtual fija y nombre DNS permanente.

(Nota: Se restituye `DB_HOST: catalogo-db` en el archivo `k8s/pod-catalogo-api.yaml` a la espera de la creación del Service).

---

## 7. Diagnóstico de Upstream en el Frontend (`host not found in upstream`)

### Salida de `kubectl logs catalogo-frontend`

```text
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: /etc/nginx/conf.d/default.conf differs from the packaged version
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
20-envsubst-on-templates.sh: Running envsubst on /etc/nginx/templates/default.conf.template to /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2026/10/19 19:38:16 [emerg] 1#1: host not found in upstream "catalogo-api" in /etc/nginx/conf.d/default.conf:24
nginx: [emerg] host not found in upstream "catalogo-api" in /etc/nginx/conf.d/default.conf:24
```

### Comparativa: Causa en Semana 3 vs Causa en Semana 5

* Semana 3 (Docker CLI): El error ocurrió porque el contenedor se inició fuera de la red `catalogo-net` (omisión del parámetro `--network`), quedando en la red bridge por defecto donde Docker no habilita la resolución DNS embebida.

* Semana 5 (Kubernetes): El Pod se encuentra dentro de la red del clúster, pero la resolución de nombres hacia otros componentes solo se produce si existe un objeto `Service` registrado. Al existir únicamente el Pod `catalogo-api` sin un Service que lo exponga, CoreDNS responde que el host no existe y Nginx aborta el arranque.

---

## 8. Acceso Mediante `kubectl port-forward` y Efecto de `ROOT_PATH`

Comando ejecutado para abrir el túnel directo:

```bash
kubectl port-forward pod/catalogo-api 8000:8000
```

### a) Comprobación de `/health` y `/productos` mediante `curl`

* Salida de `curl -s localhost:8000/health`:

```json
{"status":"ok","db":"ok","version":"v1"}
```

* Salida de `curl -s localhost:8000/productos`:

```json
[{"sku":999666,"categoria":"Apple","creado_en":"2026-10-19T19:58:12.826000","descripcion":"Clásico de 4.7\" con Touch ID y cuerpo de aluminio.","imagen_hash":"7e23251b780a76a1","imagen_tipo":"image/jpeg","nombre":"Apple iPhone 6 (Gold, 32 GB)","precio":44000,"stock":2,"id":"6ad67654ef14b98ce15b7a4b","imagen":true},{"sku":379182,"categoria":"Apple","creado_en":"2026-10-19T19:58:12.831000","descripcion":"Pantalla Retina HD de 5.5\", carga inalámbrica y chip A11.","imagen_hash":"27cf5a0cc72a5c03","imagen_tipo":"image/jpeg","nombre":"Apple iPhone 8 Plus (Silver, 64 GB)","precio":48000,"stock":6,"id":"6ad67654ef14b98ce15b7a51","imagen":true},{"sku":456654,"categoria":"Apple","creado_en":"2026-10-19T19:58:12.786000","descripcion":"Super Retina de 5.8\", Face ID y chip A11 Bionic.","imagen_hash":"8101c6ff7c22f74d","imagen_tipo":"image/jpeg","nombre":"Apple iPhone X (Space Gray, 256 GB)","precio":25000,"stock":3,"id":"6ad67654ef14b98ce15b7a41","imagen":true},{"sku":963852,"categoria":"Asus","creado_en":"2026-10-19T19:58:12.812000","descripcion":"Batería de 4000 mAh que funciona también como power bank.","imagen_hash":"95255abbb146483b","imagen_tipo":"image/jpeg","nombre":"Asus ZenFone Max M1 (Black, 32 GB) (3 GB RAM)","precio":84000,"stock":0,"id":"6ad67654ef14b98ce15b7a49","imagen":true},{"sku":123222,"categoria":"Honor","creado_en":"2026-10-19T19:58:12.789000","descripcion":"Pantalla FullView de 5.84\" con notch y cámara dual de 13 MP.","imagen_hash":"1e45d7360052e1e1","imagen_tipo":"image/jpeg","nombre":"Honor 9N (Midnight Black, 32 GB) (3 GB RAM)","precio":39000,"stock":11,"id":"6ad67654ef14b98ce15b7a45","imagen":true},{"sku":741852,"categoria":"Lenovo","creado_en":"2026-10-19T19:58:12.808000","descripcion":"Procesador Helio X23 de diez núcleos y Android puro.","imagen_hash":"5bf64a1fb53294cd","imagen_tipo":"image/jpeg","nombre":"Lenovo K8 Note (Venom Black, 64 GB) (4 GB RAM)","precio":36000,"stock":5,"id":"6ad67654ef14b98ce15b7a47","imagen":true},{"sku":357842,"categoria":"OPPO","creado_en":"2026-10-19T19:58:12.785000","descripcion":"Lector de huellas bajo la pantalla AMOLED y cámara frontal de 25 MP.","imagen_hash":"24b3652bc891267c","imagen_tipo":"image/jpeg","nombre":"OPPO K1 (Piano Black, 64 GB) (4 GB RAM)","precio":65000,"stock":6,"id":"6ad67654ef14b98ce15b7a3f","imagen":true},{"sku":852369,"categoria":"Xiaomi","creado_en":"2026-10-19T19:58:12.760000","descripcion":"Pantalla 5.99\" HD+, cámara dual de 12 MP y batería de 3080 mAh.","imagen_hash":"3926b6235eecc7b3","imagen_tipo":"image/jpeg","nombre":"Redmi Y2 (Black, 32 GB) (3 GB RAM)","precio":25000,"stock":14,"id":"6ad67654ef14b98ce15b7a3b","imagen":true},{"sku":848484,"categoria":"Samsung","creado_en":"2026-10-19T19:58:12.830000","descripcion":"Super AMOLED de 6.3\" y cámara dual de 16 MP + 24 MP.","imagen_hash":"6e0b6f7b0f8514c0","imagen_tipo":"image/jpeg","nombre":"Samsung Galaxy A8 Star (White, 64 GB) (6 GB RAM)","precio":78000,"stock":8,"id":"6ad67654ef14b98ce15b7a4f","imagen":true},{"sku":332211,"categoria":"Samsung","creado_en":"2026-10-19T19:58:12.828000","descripcion":"Primer smartphone con cuatro cámaras traseras y pantalla de 6.3\".","imagen_hash":"97b0b86fea2f5dc7","imagen_tipo":"image/jpeg","nombre":"Samsung Galaxy A9 (Bubblegum Pink, 128 GB)","precio":97000,"stock":4,"id":"6ad67654ef14b98ce15b7a4d","imagen":true},{"sku":159753,"categoria":"Samsung","creado_en":"2026-10-19T19:58:12.782000","descripcion":"Batería de 5000 mAh con carga rápida y pantalla Infinity-V de 6.3\".","imagen_hash":"178a6127bb70f3cb","imagen_tipo":"image/jpeg","nombre":"Samsung Galaxy M20 (Charcoal Black, 4+64GB)","precio":58000,"stock":9,"id":"6ad67654ef14b98ce15b7a3d","imagen":true},{"sku":358426,"categoria":"Vivo","creado_en":"2026-10-19T19:58:12.788000","descripcion":"Cámara frontal pop-up de 32 MP y triple cámara trasera de 48 MP.","imagen_hash":"11814686e87c1818","imagen_tipo":"image/jpeg","nombre":"Vivo V15 Pro (Topaz Blue, 128 GB) (6 GB RAM)","precio":55000,"stock":7,"id":"6ad67654ef14b98ce15b7a43","imagen":true}]
```

### b) Análisis del esquema OpenAPI, Swagger UI y efecto de `ROOT_PATH`

Al consultar `/api/openapi.json`, se evidencia la presencia de:

```json
"servers":[{"url":"/api"}]
```

* **Comportamiento esperado según el diseño original:** La variable `ROOT_PATH=/api` fue configurada bajo la premisa de que Nginx actuaría como reverse proxy, recibiendo solicitudes bajo `/api` y reescribiéndolas hacia la raíz de FastAPI (`/`). Sin la presencia de Nginx (mediante `port-forward`), el cliente web interactúa directamente con el puerto 8000 sin una capa de reescritura intermedia.
* **Comportamiento observado:** En la versión implementada con FastAPI, el framework absorbe el prefijo definido en `root_path` y permite resolver las peticiones a la API tanto con como sin prefijo, posibilitando que Swagger UI renderice y ejecute consultas hacia `/api/productos`.
* **Naturaleza de `port-forward`:** Se confirma que `port-forward` es una herramienta exclusiva para depuración e inspección directa en fases de desarrollo; no provee balanceo de carga, persistencia ante recreación del Pod ni reemplaza a un objeto `Service` o `Ingress` para publicación formal.

---

## 9. Mortalidad de los Pods y Ausencia de Controladores

Comando de eliminación ejecutado:

```bash
kubectl delete pod catalogo-api
```

### Verificación del estado posterior (`kubectl get pods`)

```text
NAME                READY   STATUS    RESTARTS         AGE
catalogo-db         1/1     Running   0                67m
catalogo-frontend   0/1     Error     13 (5m45s ago)   44m
```

### Conclusión técnica

Un Pod desnudo (*naked pod*) carece de mecanismos de supervisión o autorecuperación. El plano de control de Kubernetes no resucita un Pod eliminado manualmente ni recrea instancias tras la caída de un nodo si no se encuentran administradas por un controlador de nivel superior (como un `Deployment` o un `ReplicaSet`).

---

## 10. Cheatsheet de Comandos `kubectl` Utilizados

1. `kubectl config current-context`: Informa el clúster y las credenciales activas en la configuración local de `kubeconfig` antes de interactuar.

2. `kubectl create namespace <nombre>`: Da de alta un nuevo espacio de nombres para aislar lógicamente los recursos del proyecto.

3. `kubectl config set-context --current --namespace=<nombre>`: Establece el namespace por defecto para el contexto en uso, omitiendo la necesidad del parámetro `-n` en llamadas posteriores.

4. `kubectl run <nombre> --image=<imagen> --dry-run=client -o yaml`: Simula la ejecución de un contenedor y genera el manifiesto declarativo YAML estándar en la salida sin aplicar cambios en el clúster.

5. `kubectl apply -f <archivo.yaml>`: Aplica de forma declarativa la configuración de los recursos descritos en el archivo manifiesto.

6. `kubectl get pods -o wide`: Lista los Pods del namespace brindando detalles extendidos: IP asignada dentro del clúster y nodo físico en el que corren.

7. `kubectl describe pod <nombre>`: Muestra el detalle exhaustivo del recurso, incluyendo condiciones de estado, contenedores y la lista cronológica de eventos del kubelet.

8. `kubectl logs <nombre> --previous`: Obtiene los registros de la consola del contenedor que finalizó en la ejecución inmediatamente anterior para diagnosticar fallas de arranque o excepciones.

9. `kubectl port-forward pod/<nombre> <puerto_local>:<puerto_pod>`: Crea un túnel TCP directo desde la máquina local hacia un puerto interno específico de un Pod sin exponerlo en la red del clúster.

10. `kubectl delete pod <nombre>`: Detiene y elimina definitivamente una instancia de Pod dentro del namespace.

---

## 11. Comparativa: Docker Compose vs Manifiestos de Pods Sueltos

| Aspecto de Infraestructura | Docker Compose (`docker-compose.yml`) | Pods Sueltos (`k8s/*.yaml`) |
| --- | --- | --- |
| **Resolución de Nombres DNS** | Automática: el nombre de cada servicio es resuelto por el DNS interno de Docker entre contenedores de la misma red. | Inexistente: un Pod no publica entradas resolubles en CoreDNS para otros Pods; requiere objetos `Service`. |
| **Estabilidad de Red** | Estable: la comunicación se realiza mediante el nombre de servicio independiente de la IP del contenedor. | Inestable: la dirección IP del Pod es efímera y cambia tras cada recreación o reinicio. |
| **Garantía de Resurrección** | Continua: Compose reinicia los servicios ante caídas y los gestiona como un grupo unificado. | Nula: un Pod individual es mortal; si muere o se borra, ningún componente lo recrea. |
| **Coordinación de Dependencias** | Integrada: `depends_on` con `condition: service_healthy` posterga el arranque hasta superar las sondas. | Inexistente: no hay cláusula `depends_on`; la sincronización depende de readiness probes y reintentos en la app. |
| **Gestión de Persistencia** | Directa: se define el volumen nombrado local y se monta en `/data/db` de forma automática. | Pendiente: los Pods sueltos carecen de almacenamiento persistente desacoplado (`PersistentVolumeClaim`). |
