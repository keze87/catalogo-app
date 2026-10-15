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

Al construir la cadena de conexión hacia la base propia de la aplicación (`catalogo`), es **obligatorio** especificar el parámetro de consulta `?authSource=admin`:

```text
mongodb://catalogo_user:catalogo_pass@<host>:27017/catalogo?authSource=admin
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

**Lo que apareció:** El archivo `docker-compose.yml`, los sondeos de salud integrados (`healthcheck`), las relaciones de dependencia explícitas con estado saludable (`condition: service_healthy`), el contrato unificado de variables mediante `.env` / `.env.example` y la restricción declarativa de capacidades del kernel Linux.

### Sondeos de Salud (Healthchecks)

#### `catalogo-db` (MongoDB 7)

* **Comando:** `mongosh --quiet --eval "db.adminCommand('ping').ok" || exit 1`
* **Ventaja frente a autenticación explícita:** El comando `ping` no requiere credenciales en MongoDB. Esto evita que la contraseña viaje en la definición del comando o quede expuesta en las herramientas de inspección (`docker inspect` o `ps`).
* **Limitaciones y lo que NO verifica:** Un estado `healthy` solo confirma que la instancia de MongoDB está activa y respondiendo a comandos de red básicos. **No prueba** que el usuario `catalogo_user` haya sido creado ni que la contraseña configurada sea correcta. Si el volumen `catalogo-db-data` ya fue inicializado previamente con otras credenciales y se cambia la clave en el `.env`, la base estará `healthy`, pero la API fallará con `Authentication failed`.

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

La siembra del catálogo se implementó mediante una operación de actualización con inserción (*upsert*) basada en el nombre del producto. Al ejecutar `docker compose up -d --force-recreate catalogo-api`, el proceso de siembra se re-ejecuta sin duplicar registros. Esta idempotencia es una precondición indispensable para cuando réplicas simultáneas de la API iniciarán en paralelo y ejecutarán la siembra concurrentemente.

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
