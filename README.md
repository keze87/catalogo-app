# catalogo-app — Proyecto Final (Variante C)

Aplicación de catálogo de productos con imágenes compuesta por tres capas (**React + Vite servido por Nginx · Python + FastAPI · MongoDB 7**), contenerizada con builds multietapa, orquestada localmente con Docker Compose y desplegada en Kubernetes mediante manifiestos individuales y un chart de Helm parametrizado.

## Artefactos del repositorio

| Artefacto | Ubicación | Descripción |
| --- | --- | --- |
| Dockerfiles multietapa | `backend/Dockerfile`, `frontend/Dockerfile` | Imágenes optimizadas sin herramientas de compilación y configuración en tiempo de ejecución. |
| Orquestación local | `docker-compose.yml` | Las tres capas con red aislada `catalogo-net`, volumen persistente y healthchecks. |
| Manifiestos de Kubernetes | `k8s/` | Los **12 objetos** exactos de la arquitectura en archivos individuales. |
| Chart de Helm | `catalogo-app/` | `Chart.yaml`, `values.yaml` y las 12 plantillas parametrizadas. |
| Documento de arquitectura | [`ARQUITECTURA.md`](ARQUITECTURA.md) | Respuestas a las preguntas obligatorias y de la Variante C. |
| Informe de funcionamiento | [`INFORME.md`](INFORME.md) | Evidencia del despliegue funcionando, HPA, Helm y aislamiento de Pods. |
| Guías paso a paso | [`tp/`](tp/) | Paso a paso de los trabajos prácticos semanales (`tp1` a `tp8`). |

> [!NOTE]
> **Credenciales de laboratorio:** Los archivos `docker-compose.yml`, `k8s/secret-catalogo-db.yaml` y `catalogo-app/values.yaml` incluyen credenciales de laboratorio (`catalogo_user` / `catalogo_pass`) únicamente con fines académicos y de reproducibilidad. En un entorno de producción real nunca deben versionarse credenciales en el repositorio; el enfoque recomendado se detalla en [`ARQUITECTURA.md`](ARQUITECTURA.md#4-qué-quedaría-pendiente-para-un-entorno-de-producción-real).

## Cómo levantar el proyecto

### Camino 1: Docker Compose (en una sola máquina)

Requisitos: Docker Engine y Docker Compose v2.

```bash
# 1. Construir las imágenes y levantar las tres capas en segundo plano
docker compose up -d --build

# 2. Verificar que los tres contenedores estén en estado healthy
docker compose ps

# 3. Comprobar la API a través del proxy del frontend
curl -s http://localhost:3000/api/health
# {"status":"ok","db":"ok","version":"v1"}
```

Abrir en el navegador:
- **Aplicación web:** `http://localhost:3000`
- **Documentación interactiva (Swagger UI):** `http://localhost:3000/api/docs`

Para detener el entorno:
```bash
# Detener conservando los datos de MongoDB (volumen catalogo-db-data)
docker compose down

# Detener y eliminar también el volumen persistente
docker compose down -v
```

### Camino 2: Helm sobre el clúster Kubernetes

Requisitos: `kubectl` configurado contra el clúster (con `nginx-ingress-controller` y `metrics-server` activos) y Helm v3.

```bash
# 1. Validar sintaxis del chart y confirmar que renderiza exactamente 12 objetos
helm lint ./catalogo-app
helm template catalogo-app ./catalogo-app | grep -c "^kind:"

# 2. Instalar la release en el namespace catalogo-app
helm install catalogo-app ./catalogo-app -n catalogo-app --create-namespace

# 3. Verificar los 12 objetos desplegados
kubectl get all,cm,secret,pvc,ingress,hpa -n catalogo-app
```

Configurar la resolución local del host `catalogo.local` apuntando a la `EXTERNAL-IP` del Ingress Controller:

```bash
INGRESS_IP=$(kubectl get svc -n nginx-ingress nginx-ingress-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$INGRESS_IP catalogo.local" | sudo tee -a /etc/hosts

# Verificar respuesta a través del Ingress
curl -s http://catalogo.local/api/health
```

Abrir en el navegador:
- **Aplicación web:** `http://catalogo.local`
- **Documentación interactiva (Swagger UI):** `http://catalogo.local/api/docs`

Ciclo de vida de la release con Helm:

```bash
# Actualizar la versión de la API a v2
helm upgrade catalogo-app ./catalogo-app -n catalogo-app --set api.image.tag=v2

# Consultar historial y revertir a la revisión 1
helm history catalogo-app -n catalogo-app
helm rollback catalogo-app 1 -n catalogo-app

# Desinstalar la release completa
helm uninstall catalogo-app -n catalogo-app
```

*(Alternativamente, los 12 manifiestos planos pueden aplicarse de forma individual desde `k8s/` mediante `kubectl create namespace catalogo-app && kubectl apply -f k8s/`.)*

---

# Arquitectura

```text
navegador ──► catalogo-frontend ──────► catalogo-api ──────► catalogo-db
              React + Vite              Python + FastAPI     MongoDB 7
              nginx :8080               :8000                :27017
              (host 3000)               (host 8000, solo     (sin puerto
                                         depuración)          publicado)
```

| Capa | Imagen | Puerto interno | Puerto publicado |
| --- | --- | --- | --- |
| `catalogo-frontend` | React + Vite servido por nginx | 8080 | 3000 |
| `catalogo-api` | Python + FastAPI | 8000 | 8000 (solo depuración) |
| `catalogo-db` | MongoDB 7 | 27017 | — |

El frontend es el único punto de entrada: nadie le habla a la base directamente, y a la API le habla el frontend.

## Consideraciones de Base de Datos y Conexión

### Autenticación y `authSource`

El usuario administrador inicial (`catalogo_user`), configurado mediante las variables de entorno `MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD` de la imagen oficial `mongo:7`, reside físicamente en la base de datos de autenticación del sistema: `admin`.

Al construir la cadena de conexión hacia la base propia de la aplicación (`catalogodb`), es **obligatorio** especificar el parámetro de consulta `?authSource=admin`:

```text
mongodb://catalogo_user:catalogo_pass@<host>:27017/catalogodb?authSource=admin
```

## Configuración de Contenedores e Imágenes

### Variable `NGINX_ENVSUBST_FILTER` en Frontend

La imagen del frontend utiliza la funcionalidad de sustitución de variables de entorno de Nginx (`envsubst`) al momento de arrancar el contenedor.

Se define de manera obligatoria:

```env
NGINX_ENVSUBST_FILTER=^API_
```

**Justificación:** Sin este filtro de expresión regular, el script de inicialización de Nginx reemplaza indiscriminadamente todas las apariciones del signo `$` en las plantillas de configuración, incluyendo variables internas de Nginx como `$uri`, `$host` o `$proxy_add_x_forwarded_for`. Esto ocasiona que dichas variables queden vacías y generen una configuración corrupta e inválida en silencio.

### Comparativa de Imágenes y Optimización

| Imagen | Tag | Base | Tamaño | Reducción vs Naive | Justificación de la decisión técnica |
| --- | --- | --- | --- | --- | --- |
| `catalogo-api` | `naive` | `python:3.12` | 1.71GB | 0% | Etapa única con sistema base completo y herramientas de compilación innecesarias. |
| `catalogo-api` | `v1` | `python:3.12-slim` | 246MB | 85% | Uso de base `slim`, build multi-etapa e instalación con `--user --no-cache-dir` descartando compiladores del sistema. |
| `catalogo-frontend` | `v1` | `nginxinc/nginx-unprivileged:1.27-alpine` | 73.9MB | N/A | Multi-etapa que desecha Node.js, `npm` y `node_modules`, copiando solo los artefactos estáticos generados en `dist/`. |

### Red aislada y aislamiento de la Base de Datos (`catalogo-db`)

El contenedor de la base de datos se ejecuta sin publicar ningún puerto hacia el anfitrión (se omite la bandera `-p`):

```bash
docker run -d \
  --name catalogo-db \
  --network catalogo-net \
  -v catalogo-db-data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=catalogo_user \
  -e MONGO_INITDB_ROOT_PASSWORD=catalogo_pass \
  mongo:7
```

**Justificación:** No se publica ningún puerto con `-p` hacia el anfitrión debido al principio de mínimo privilegio y defensa en profundidad: la base de datos es la capa más interna y confidencial de la arquitectura, por lo que únicamente debe ser accesible para los contenedores que coexisten en la red aislada `catalogo-net`. Al no exponer el puerto `27017` en el *host*, se elimina la superficie de ataque desde el exterior del entorno Docker.

## Despliegue Manual y Secuencia de Inicialización

### Secuencia Estricta

1. `docker network create catalogo-net`
2. `docker volume create catalogo-db-data`
3. Iniciar `catalogo-db` (esperar a que el motor complete su inicialización)
4. Iniciar `catalogo-api` (requiere resolución y puerto activo de catalogo-db)
5. Iniciar `catalogo-frontend` (requiere resolución activa de catalogo-api en el DNS)

### Matriz de Errores por Alteración del Orden

* **Iniciar componentes antes de la red:** Docker interrumpe la creación informando que la red no existe.
* **Iniciar API antes de la Base de Datos:** La API no puede establecer el socket inicial con MongoDB, arrojando errores de conexión o cerrando el proceso.
* **Iniciar Frontend antes de la API:** Nginx intenta resolver el nombre `catalogo-api` en su inicialización. Al fallar la resolución DNS, aborta con `[emerg] ... host not found in upstream "catalogo-api"` y el contenedor termina en `Exited (1)`.

---

## Orquestación con Docker Compose

### Comparativa: Docker CLI vs Docker Compose

| Aspecto | `docker run` | `docker-compose.yml` |
| --- | --- | --- |
| **Definición** | Imperativa, mediante comandos independientes en la CLI. | Declarativa, centralizada e infraestructura como código. |
| **Redes y Volúmenes** | Creación previa explícita (`docker network create`, `docker volume create`). | Declarados en el bloque superior (`networks:`, `volumes:`). |
| **Sincronización** | Coordinación manual por tiempos de espera empíricos. | Automatizada mediante `healthcheck` y `depends_on` condicional. |
| **Mínimo Privilegio** | Banderas de aislamiento aplicadas manualmente por comando. | Control declarativo granular de `cap_drop` y `cap_add`. |

**Lo que desapareció:** La necesidad de ejecutar comandos en un orden estricto, los retardos manuales entre contenedores, la creación imperativa de la red `catalogo-net` y los parámetros extensos por línea de comandos.

**Lo que apareció:** El archivo `docker-compose.yml`, los sondeos de salud integrados (`healthcheck`), las relaciones de dependencia explícitas con estado saludable (`condition: service_healthy`), la declaración centralizada de variables de entorno en cada servicio (con `.env.example` como referencia de contrato) y la restricción declarativa de capacidades del kernel Linux.

### Sondeos de Salud (Healthchecks)

#### `catalogo-db` (MongoDB 7)

* **Comando:** `mongosh --quiet --eval "db.adminCommand('ping').ok" || exit 1`
* **Ventaja frente a autenticación explícita:** El comando `ping` no requiere credenciales en MongoDB. Esto evita que la contraseña viaje en la definición del comando o quede expuesta en las herramientas de inspección (`docker inspect` o `ps`).
* **Limitaciones y lo que NO verifica:** Un estado `healthy` solo confirma que la instancia de MongoDB está activa y respondiendo a comandos de red básicos. **No prueba** que el usuario `catalogo_user` haya sido creado ni que la contraseña configurada sea correcta. Si el volumen `catalogo-db-data` ya fue inicializado previamente con otras credenciales y se cambia la clave en `docker-compose.yml`, la base estará `healthy`, pero la API fallará con `Authentication failed`.

#### `catalogo-api` (Python / FastAPI)

* **Comando:** `python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"`
* **Justificación:** Se utiliza el módulo estándar `urllib.request` nativo de Python. Instalar herramientas adicionales como `curl` en la imagen runtime de producción únicamente para ejecutar el *healthcheck* incrementaría el tamaño del contenedor y expondría librerías innecesarias, violando el principio de superficie de ataque mínima.

#### `catalogo-frontend` (Nginx / React)

* **Comando:** `wget --no-verbose --tries=1 --spider http://127.0.0.1:8080/healthz`
* **Justificación:** Emplea `wget`, ejecutable ya presente de manera nativa en la distribución Alpine / BusyBox del contenedor, invocando el *endpoint* dedicado de salud configurado en Nginx.

### Control de Dependencias (`depends_on` con `condition`)

Las dependencias entre servicios se definieron mediante estados de salud explícitos (`service_healthy`), abordando dos problemáticas no equivalentes:

* **`catalogo-api` respecto de `catalogo-db`:**
La API requiere establecer una conexión activa y ejecutar la siembra de datos al iniciar. La forma corta (`depends_on: [catalogo-db]`) únicamente aguarda el inicio del contenedor, no la disponibilidad de MongoDB para aceptar sockets. La API reintenta la conexión durante aproximadamente 30 segundos; este mecanismo de reintento en código es lo único que subsistirá en Kubernetes, donde no existe la directiva `depends_on`.
* **`catalogo-frontend` respecto de `catalogo-api`:**
Sin `depends_on`, el frontend iniciaría en paralelo a la API. Al arrancar Nginx, la falta de resolución DNS de `catalogo-api` provocaría la caída inmediata del servicio con el error `host not found in upstream`. Aunque la forma corta de `depends_on` basta para resolver el nombre en el DNS, la condición `service_healthy` garantiza además que el frontend comience a recibir tráfico cuando la API ya se encuentra respondiendo, evitando errores HTTP 502 durante los primeros segundos.

### Ciclo de Vida de los Datos e Idempotencia

#### Diferencia entre `docker compose down` y `docker compose down -v`

* `docker compose down`: Detiene y remueve los contenedores y la red `catalogo-net`, **preservando intacto** el volumen nombrado `catalogo-db-data`. Los datos y el catálogo persisten entre reinicios.
* `docker compose down -v`: Elimina contenedores, redes y **destruye los volúmenes persistentes**. En el subsiguiente inicio, la base de datos iniciará desde cero y ejecutará nuevamente el proceso de inicialización.

#### Idempotencia en la Siembra de Datos

La siembra del catálogo se implementó mediante una operación de actualización con inserción (*upsert*) basada en el `sku` único del producto. Al ejecutar `docker compose up -d --force-recreate catalogo-api`, el proceso de siembra se re-ejecuta sin duplicar registros. Esta idempotencia es una precondición indispensable para cuando réplicas simultáneas de la API iniciarán en paralelo y ejecutarán la siembra concurrentemente.

### Auditoría de Seguridad y Mínimo Privilegio

#### Mínimo Privilegio y Control de Capabilities (`cap_drop` / `cap_add`)

Se aplicó la remoción total de capacidades (`cap_drop: [ALL]`) en los tres servicios:

1. **`catalogo-api` y `catalogo-frontend`:** Funcionan correctamente sin capacidades del kernel debido a que ejecutan sus procesos principales bajo usuarios sin privilegios (`appuser` UID 1000 y `nginx` UID 101 respectivamente).
2. **`catalogo-db`:** Al aplicar únicamente `cap_drop: [ALL]`, el servicio falla durante la fase de inicialización arrojando errores de denegación de acceso al intentar validar los archivos del volumen persistente:

```text
catalogo-db  | find: '/data/db/.mongodb': Permission denied
catalogo-db  | find: '/data/db/diagnostic.data': Permission denied
catalogo-db  | find: '/data/db/journal': Permission denied
catalogo-db exited with code 1
```

**Causa y Solución:** El script de entrada de la imagen oficial (`docker-entrypoint.sh`) inspecciona y ajusta la propiedad de los directorios internos en `/data/db` antes de realizar el cambio de usuario. Para permitir la ejecución completa de este proceso respetando el mínimo privilegio, se agregaron únicamente las capacidades requeridas mediante `cap_add`:

* **`CHOWN`:** Permite cambiar la titularidad y grupo de los archivos y directorios dentro de `/data/db`.
* **`DAC_OVERRIDE`:** Permite ignorar las restricciones de permisos de lectura, escritura y ejecución al escanear los archivos del volumen.
* **`FOWNER`:** Permite modificar atributos y permisos en archivos que no son propiedad del UID runtime actual.
* **`SETUID`:** Permite cambiar la identidad del proceso (UID) de `root` al usuario `mongodb` (UID 999).
* **`SETGID`:** Permite cambiar el grupo del proceso (GID) al grupo `mongodb` (GID 999).

#### Matriz de Auditoría de Seguridad

| Servicio | Usuario Runtime | Puertos Expuestos | Capabilities Mínimas | Escaneo Vulnerabilidades (Trivy / Scout) |
| --- | --- | --- | --- | --- |
| `catalogo-db` | `mongodb` (UID 999) | Ninguno (sólo interno) | `CHOWN`, `DAC_OVERRIDE`, `FOWNER`, `SETUID`, `SETGID` | 135 |
| `catalogo-api` | `appuser` (UID 1000) | `8000:8000` *(Solo depuración; será interno)* | Ninguna (`cap_drop: [ALL]`) | 183 |
| `catalogo-frontend` | `nginx` (UID 101) | `3000:8080` | Ninguna (`cap_drop: [ALL]`) | 159 |

---

## Kubernetes

### Comparativa causa `host not found in upstream`: Semana 3 vs Semana 5

* Semana 3 (Docker CLI): El error ocurrió porque el contenedor se inició fuera de la red `catalogo-net` (omisión del parámetro `--network`), quedando en la red bridge por defecto donde Docker no habilita la resolución DNS embebida.

* Semana 5 (Kubernetes): El Pod se encuentra dentro de la red del clúster, pero la resolución de nombres hacia otros componentes solo se produce si existe un objeto `Service` registrado. Al existir únicamente el Pod `catalogo-api` sin un Service que lo exponga, CoreDNS responde que el host no existe y Nginx aborta el arranque.

## Cheatsheet de Comandos `kubectl` Utilizados

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

## Verificación de Selectors y Endpoints

Comando de verificación ejecutado:

```bash
kubectl get endpoints -n catalogo-app
```

**Salida registrada:**

```text
NAME                     ENDPOINTS                               AGE
catalogo-api             10.244.0.27:8000,10.244.0.28:8000       5m
catalogo-db              10.244.0.26:27017                       5m
catalogo-frontend        10.244.0.24:8080,10.244.0.29:8080       5m
```

* **Análisis:** La correspondencia entre el campo `spec.selector` de cada Service y `metadata.labels` de las plantillas de los Deployments vincula dinámicamente las direcciones IP efímeras de las réplicas en ejecución. Si hubiese discrepancia en las etiquetas, la columna `ENDPOINTS` mostraría `<none>`.

## Service NodePort de Depuración (`catalogo-api-nodeport`)

Para verificar el backend directamente sin la intermediación de Nginx, se definió un Service auxiliar en el puerto 30502:

```bash
curl -s http://<IP-del-nodo>:30502/health
# {"status":"ok","db":"ok","version":"v1"}
```

> **Aclaración explícita:** El objeto `catalogo-api-nodeport` fue creado con propósitos exclusivos de depuración y prueba para este práctico. **No formará parte del chart de Helm de la Semana 8**, donde el backend permanecerá exclusivamente accesible de forma interna a través de su Service `ClusterIP`.

## Limitación Conocida: Persistencia en la Base de Datos

### Comprobación de pérdida de datos

1. Se forzó la eliminación del Pod de la base de datos:

```bash
kubectl delete pod -l app=catalogo-db -n catalogo-app
```


2. Al recrearse el Pod, la consulta a `/productos` devolvió un array vacío (`[]`). El producto cargado manualmente en el Punto 3 se perdió de forma definitiva.
3. Se reinició el Deployment de la API para disparar la rutina de siembra:

```bash
kubectl rollout restart deployment/catalogo-api -n catalogo-app
```

4. Al reiniciar la API, reaparecieron únicamente los 12 productos de demostración (`SEED_DEMO=true`), pero no el producto manual.

### Declaración de limitación

El objeto `catalogo-db` opera actualmente como un `Deployment` respaldado únicamente por el almacenamiento efímero del nodo. **Esta constituye una limitación conocida y explícita del TP6**, la cual será subsanada en la Semana 7 mediante la desacoplación del almacenamiento con `PersistentVolume` (PV) y `PersistentVolumeClaim` (PVC).

---

## Configuración Externalizada, Persistencia e Ingress (TP7)

### Externalización de Configuración (`ConfigMap` y `Secret`)

Ningún valor de configuración o credencial permanece escrito literalmente en los manifiestos de `Deployment`:

* **`catalogo-api-config` (`ConfigMap`):** Define `DB_HOST`, `DB_PORT`, `DB_NAME`, `ROOT_PATH` (`/api`) y `SEED_DEMO` (`"true"`). No incluye `APP_VERSION`, ya que esa variable viene fijada en la imagen durante el `docker build`.
* **`catalogo-frontend-config` (`ConfigMap`):** Define `API_HOST` y `API_PORT`.
* **`catalogo-db-secret` (`Secret`):** Almacena `MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD`. Se reutiliza tanto en MongoDB (`envFrom`) como en FastAPI mediante mapeo con `valueFrom.secretKeyRef` hacia `DB_USER` y `DB_PASSWORD`.
  * **Seguridad:** Los `Secret` estándar de Kubernetes codifican sus valores en Base64 (reversible, no cifrado). Para proteger credenciales en reposo se requiere habilitar *Encryption at Rest* en `etcd` o utilizar un gestor de secretos externo (HashiCorp Vault, Sealed Secrets, External Secrets Operator).

### Decisión Arquitectónica: Cadena de Conexión a MongoDB

Se optó por entregar las **piezas por separado** (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`) en lugar de una única variable `MONGO_URI`:

1. **Reutilización sin duplicación:** Un único `Secret` (`catalogo-db-secret`) alimenta tanto a MongoDB oficial (`MONGO_INITDB_ROOT_*`) como a FastAPI.
2. **Autenticación automática:** FastAPI ensambla la URI internamente y anexa `?authSource=admin` por sí misma, evitando errores `Authentication failed` por omisión manual.

### Persistencia de Datos (`catalogo-db-pvc`)

Se resolvió la limitación declarada en el TP6 incorporando `catalogo-db-pvc` (`1Gi`, `ReadWriteOnce`, StorageClass por defecto) montado en `/data/db`:

* **Ausencia de `subPath`:** A diferencia de MySQL, MongoDB tolera que `/data/db` contenga subdirectorios iniciales (como `lost+found`), por lo que no requiere `subPath`.
* **Comprobación válida:** La siembra del catálogo de demostración (`SEED_DEMO=true`) es idempotente y se ejecuta cada vez que arranca la API, por lo que su reaparición tras recrear el Pod constituye un falso positivo. La persistencia efectiva se verificó cargando manualmente un producto con imagen y comprobando su supervivencia tras eliminar el Pod de `catalogo-db`.

### Exposición mediante Ingress (`catalogo-app-ingress`)

* **Regla única (`/`) hacia `catalogo-frontend:8080` bajo `catalogo.local`:** No se agrega una regla `/api` en el `Ingress` porque el Nginx interno del frontend ya realiza el proxy inverso hacia `catalogo-api:8000`; duplicar la regla crearía dos fuentes de verdad y rompería la paridad con Docker Compose.
* **Verificación válida vs. falso positivo SPA:** La ruta `/productos` devuelve `200 OK` con el `index.html` de React por la directiva `try_files $uri $uri/ /index.html;`. La verificación real del backend a través del Ingress se realiza sobre `/api/productos` (JSON) y `/api/docs` (Swagger UI, confirmando la llegada de `ROOT_PATH=/api`).

### Inventario de Objetos en el Clúster (11 de 12)

| # | Nombre del Objeto | Tipo (`Kind`) | Función |
| --- | --- | --- | --- |
| 1 | `catalogo-db` | `Deployment` | Gestiona la réplica única de MongoDB 7 con almacenamiento persistente. |
| 2 | `catalogo-api` | `Deployment` | Mantiene las réplicas sin estado del backend FastAPI. |
| 3 | `catalogo-frontend` | `Deployment` | Administra las réplicas de Nginx con el build estático de React y proxy `/api`. |
| 4 | `catalogo-db` | `Service` (`ClusterIP`) | Provee resolución DNS interna en el puerto `27017` hacia MongoDB. |
| 5 | `catalogo-api` | `Service` (`ClusterIP`) | Balancea internamente las peticiones HTTP en el puerto `8000` hacia la API. |
| 6 | `catalogo-frontend` | `Service` (`NodePort`) | Expone el frontend en el puerto `30302` de los nodos y recibe tráfico del Ingress. |
| 7 | `catalogo-api-config` | `ConfigMap` | Externaliza los parámetros no sensibles del backend. |
| 8 | `catalogo-frontend-config` | `ConfigMap` | Externaliza el host y puerto de destino de la API para Nginx. |
| 9 | `catalogo-db-secret` | `Secret` | Almacena las credenciales de administrador de MongoDB. |
| 10 | `catalogo-db-pvc` | `PersistentVolumeClaim` | Solicita `1Gi` (`ReadWriteOnce`) para persistir `/data/db`. |
| 11 | `catalogo-app-ingress` | `Ingress` | Enruta el tráfico HTTP de `catalogo.local` (puerto 80) hacia `catalogo-frontend`. |

---

## Autoescalado Horizontal y Empaquetado con Helm (TP8)

### Autoescalado de la API (`catalogo-api-hpa`)

* **Declaración de recursos en `catalogo-api`:** `resources.requests` (`cpu: 100m`, `memory: 128Mi`) y `limits` (`cpu: 300m`, `memory: 256Mi`). El `HorizontalPodAutoscaler` calcula el porcentaje de uso dividiendo el consumo reportado por `metrics-server` sobre `requests.cpu`.
* **Configuración del HPA:** `minReplicas: 2`, `maxReplicas: 6`, objetivo `averageUtilization: 50` sobre CPU.
* **HPA único en la arquitectura:**
  * `catalogo-db` (MongoDB) no se escala horizontalmente mediante un `Deployment` porque cada réplica requeriría su propio almacenamiento sin *replica set* configurado.
  * `catalogo-frontend` (Nginx) sirve estáticos y actúa como reverse proxy ligero; la carga de cómputo y lectura/entrega de imágenes recae sobre `catalogo-api` (*stateless*).
* **Transición de 1 a 3 réplicas (*stateless*):** Al balancear múltiples réplicas de la API, deja de estar garantizada la afinidad con el sistema de archivos local o la memoria RAM de un Pod individual; por ello todo el estado (incluidas las imágenes binarias) reside en MongoDB sobre su `PersistentVolumeClaim`.

### Despliegue con Helm (`catalogo-app/`)

El proyecto consolida los **12 objetos exactos** en el chart `./catalogo-app` (excluyendo `catalogo-api-nodeport`, usado solo para depuración en el TP6), parametrizando en `values.yaml` imágenes/tags, réplicas, recursos, umbrales del HPA, almacenamiento y host del Ingress:

```bash
# Validar sintaxis y renderizado (12 objetos)
helm lint ./catalogo-app
helm template catalogo-app ./catalogo-app | grep -c "^kind:"

# Instalar release
helm install catalogo-app ./catalogo-app -n catalogo-app --create-namespace

# Actualizar versión de la API y revertir si es necesario
helm upgrade catalogo-app ./catalogo-app -n catalogo-app --set api.image.tag=v2
helm history catalogo-app -n catalogo-app
helm rollback catalogo-app 1 -n catalogo-app
```

### Inventario Final de Objetos en el Clúster (12 de 12)

| # | Nombre del Objeto | Tipo (`Kind`) | Función |
| --- | --- | --- | --- |
| 1 | `catalogo-db` | `Deployment` | Gestiona la réplica única de MongoDB 7 con almacenamiento persistente. |
| 2 | `catalogo-api` | `Deployment` | Mantiene las réplicas sin estado del backend FastAPI con `resources.requests`. |
| 3 | `catalogo-frontend` | `Deployment` | Administra las réplicas de Nginx con el build estático de React y proxy `/api`. |
| 4 | `catalogo-db` | `Service` (`ClusterIP`) | Provee resolución DNS interna en el puerto `27017` hacia MongoDB. |
| 5 | `catalogo-api` | `Service` (`ClusterIP`) | Balancea internamente las peticiones HTTP en el puerto `8000` hacia la API. |
| 6 | `catalogo-frontend` | `Service` (`NodePort`) | Expone el frontend en el puerto `30302` de los nodos y recibe tráfico del Ingress. |
| 7 | `catalogo-api-config` | `ConfigMap` | Externaliza los parámetros no sensibles del backend. |
| 8 | `catalogo-frontend-config` | `ConfigMap` | Externaliza el host y puerto de destino de la API para Nginx. |
| 9 | `catalogo-db-secret` | `Secret` | Almacena las credenciales de administrador de MongoDB. |
| 10 | `catalogo-db-pvc` | `PersistentVolumeClaim` | Solicita `1Gi` (`ReadWriteOnce`) para persistir `/data/db`. |
| 11 | `catalogo-app-ingress` | `Ingress` | Enruta el tráfico HTTP de `catalogo.local` (puerto 80) hacia `catalogo-frontend`. |
| 12 | `catalogo-api-hpa` | `HorizontalPodAutoscaler` | Escala `catalogo-api` entre 2 y 6 réplicas según utilización de CPU (50%). |
