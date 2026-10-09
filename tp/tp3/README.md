# Informe - Trabajo Práctico N° 3

**Asignatura:** Contenedores, Docker y Orquestación con Kubernetes
**Variante:** C - catalogo-app [React + Vite, Python + FastAPI, MongoDB 7]
**Alumno:** César Ezequiel Herrera

---

## 1. Creación de Red y Volumen

Comandos ejecutados:

```bash
docker network create catalogo-net
docker volume create catalogo-db-data
```

### Salida de `docker network ls`

```text
NETWORK ID     NAME           DRIVER    SCOPE
d8e021c438d5   bridge         bridge    local
8a4143a9e57f   catalogo-net   bridge    local
4bc0b8594185   host           host      local
e34c83a861e6   none           null      local
```

### Salida de `docker volume ls`

```text
DRIVER    VOLUME NAME
local     2c4822ad6435be7edbd00fcf813a74db6be3bab58d63153f7323aa661b994e8c
local     08e0a6888294e5976dd085c8af5fb9cfb760a6b185513135543b5775824a402a
local     9a6cb3a45e2ef929a9176a13ee4250178ba2e609e34df4aefb1d368d60272494
local     97c550cbdf5412f604fab5d01a23b6303b59e19c02ce5deb737609b91b41034b
local     145a30eda3afe90a2d6e20d8f4706cf69508d97e6571a104fe6dd25e008b8005
local     3966a44e35f2224d8042c4ff330066f98547e5ede302ea6984fb62ebff7714b8
local     a08aefb6ee04b31eb40cbb7b13d291a00829fafabd6532018bc2e37f5da466d6
local     catalogo-db-data
```

---

## 2. Ejecución del Contenedor de Base de Datos (`catalogo-db`)

Comando de inicialización:

```bash
docker run -d \
  --name catalogo-db \
  --network catalogo-net \
  -v catalogo-db-data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=catalogo_user \
  -e MONGO_INITDB_ROOT_PASSWORD=catalogo_pass \
  mongo:7
```

### Registro de disponibilidad del motor (`docker logs catalogo-db`)

```json
{"t":{"$date":"2026-10-08T20:19:12.603+00:00"},"s":"I",  "c":"NETWORK",  "id":23016,   "ctx":"listener","msg":"Waiting for connections","attr":{"port":27017,"ssl":"off"}}
```

### Justificación de la ausencia de publicación de puertos (`-p`)

No se publica ningún puerto con `-p` hacia el anfitrión debido al principio de mínimo privilegio y defensa en profundidad: la base de datos es la capa más interna y confidencial de la arquitectura, por lo que únicamente debe ser accesible para los contenedores que coexisten en la red aislada `catalogo-net`. Al no exponer el puerto `27017` en el *host*, se elimina la superficie de ataque desde el exterior del entorno Docker.

---

## 3. Ejecución y Verificación de la API (`catalogo-api`)

Comando de despliegue:

```bash
docker run -d \
  --name catalogo-api \
  --network catalogo-net \
  -e DB_HOST=catalogo-db \
  -e DB_PORT=27017 \
  -e DB_NAME=catalogodb \
  -e DB_USER=catalogo_user \
  -e DB_PASSWORD=catalogo_pass \
  -e APP_VERSION=v1 \
  -e ROOT_PATH=/api \
  -e SEED_DEMO=true \
  -p 8000:8000 \
  catalogo-api:v1
```

### a) Resolución DNS interna (`getent hosts`)

```bash
docker exec catalogo-api getent hosts catalogo-db
```

**Salida obtenida:**

```text
172.18.0.2      catalogo-db
```

### b) Verificación de estado (`curl localhost:8000/health`)

```bash
curl -s http://localhost:8000/health
```

**Salida obtenida:**

```json
{
  "status": "ok",
  "db": "ok",
  "version": "v1"
}
```

### c) Cantidad de productos sembrados (`SEED_DEMO=true`)

```bash
curl -s http://localhost:8000/productos
```

**Salida / Conteo:**

```json
[{"sku":999666,"categoria":"Apple","creado_en":"2026-10-08T21:14:34.858000","descripcion":"Clásico de 4.7\" con Touch ID y cuerpo de aluminio.","imagen_hash":"7e23251b780a76a1","imagen_tipo":"image/jpeg","nombre":"Apple iPhone 6 (Gold, 32 GB)","precio":44000,"stock":2,"id":"6ac807ba828306d4e562e515","imagen":true},{"sku":379182,"categoria":"Apple","creado_en":"2026-10-08T21:14:34.860000","descripcion":"Pantalla Retina HD de 5.5\", carga inalámbrica y chip A11.","imagen_hash":"27cf5a0cc72a5c03","imagen_tipo":"image/jpeg","nombre":"Apple iPhone 8 Plus (Silver, 64 GB)","precio":48000,"stock":6,"id":"6ac807ba828306d4e562e51b","imagen":true},{"sku":456654,"categoria":"Apple","creado_en":"2026-10-08T21:14:34.854000","descripcion":"Super Retina de 5.8\", Face ID y chip A11 Bionic.","imagen_hash":"8101c6ff7c22f74d","imagen_tipo":"image/jpeg","nombre":"Apple iPhone X (Space Gray, 256 GB)","precio":25000,"stock":3,"id":"6ac807ba828306d4e562e50b","imagen":true},{"sku":963852,"categoria":"Asus","creado_en":"2026-10-08T21:14:34.857000","descripcion":"Batería de 4000 mAh que funciona también como power bank.","imagen_hash":"95255abbb146483b","imagen_tipo":"image/jpeg","nombre":"Asus ZenFone Max M1 (Black, 32 GB) (3 GB RAM)","precio":84000,"stock":0,"id":"6ac807ba828306d4e562e513","imagen":true},{"sku":123222,"categoria":"Honor","creado_en":"2026-10-08T21:14:34.855000","descripcion":"Pantalla FullView de 5.84\" con notch y cámara dual de 13 MP.","imagen_hash":"1e45d7360052e1e1","imagen_tipo":"image/jpeg","nombre":"Honor 9N (Midnight Black, 32 GB) (3 GB RAM)","precio":39000,"stock":11,"id":"6ac807ba828306d4e562e50f","imagen":true},{"sku":741852,"categoria":"Lenovo","creado_en":"2026-10-08T21:14:34.856000","descripcion":"Procesador Helio X23 de diez núcleos y Android puro.","imagen_hash":"5bf64a1fb53294cd","imagen_tipo":"image/jpeg","nombre":"Lenovo K8 Note (Venom Black, 64 GB) (4 GB RAM)","precio":36000,"stock":5,"id":"6ac807ba828306d4e562e511","imagen":true},{"sku":357842,"categoria":"OPPO","creado_en":"2026-10-08T21:14:34.853000","descripcion":"Lector de huellas bajo la pantalla AMOLED y cámara frontal de 25 MP.","imagen_hash":"24b3652bc891267c","imagen_tipo":"image/jpeg","nombre":"OPPO K1 (Piano Black, 64 GB) (4 GB RAM)","precio":65000,"stock":6,"id":"6ac807ba828306d4e562e509","imagen":true},{"sku":852369,"categoria":"Xiaomi","creado_en":"2026-10-08T21:14:34.851000","descripcion":"Pantalla 5.99\" HD+, cámara dual de 12 MP y batería de 3080 mAh.","imagen_hash":"3926b6235eecc7b3","imagen_tipo":"image/jpeg","nombre":"Redmi Y2 (Black, 32 GB) (3 GB RAM)","precio":25000,"stock":14,"id":"6ac807ba828306d4e562e505","imagen":true},{"sku":848484,"categoria":"Samsung","creado_en":"2026-10-08T21:14:34.859000","descripcion":"Super AMOLED de 6.3\" y cámara dual de 16 MP + 24 MP.","imagen_hash":"6e0b6f7b0f8514c0","imagen_tipo":"image/jpeg","nombre":"Samsung Galaxy A8 Star (White, 64 GB) (6 GB RAM)","precio":78000,"stock":8,"id":"6ac807ba828306d4e562e519","imagen":true},{"sku":332211,"categoria":"Samsung","creado_en":"2026-10-08T21:14:34.858000","descripcion":"Primer smartphone con cuatro cámaras traseras y pantalla de 6.3\".","imagen_hash":"97b0b86fea2f5dc7","imagen_tipo":"image/jpeg","nombre":"Samsung Galaxy A9 (Bubblegum Pink, 128 GB)","precio":97000,"stock":4,"id":"6ac807ba828306d4e562e517","imagen":true},{"sku":159753,"categoria":"Samsung","creado_en":"2026-10-08T21:14:34.852000","descripcion":"Batería de 5000 mAh con carga rápida y pantalla Infinity-V de 6.3\".","imagen_hash":"178a6127bb70f3cb","imagen_tipo":"image/jpeg","nombre":"Samsung Galaxy M20 (Charcoal Black, 4+64GB)","precio":58000,"stock":9,"id":"6ac807ba828306d4e562e507","imagen":true},{"sku":358426,"categoria":"Vivo","creado_en":"2026-10-08T21:14:34.854000","descripcion":"Cámara frontal pop-up de 32 MP y triple cámara trasera de 48 MP.","imagen_hash":"11814686e87c1818","imagen_tipo":"image/jpeg","nombre":"Vivo V15 Pro (Topaz Blue, 128 GB) (6 GB RAM)","precio":55000,"stock":7,"id":"6ac807ba828306d4e562e50d","imagen":true}]
```

---

## 4. Diagnóstico de Fallas: Resolución DNS vs Autenticación

### a) Fallo por nombre de host erróneo (`DB_HOST=catalogo-bd`)

```bash
docker logs catalogo-api
```

**Mensaje registrado:**

```log
[2026-10-08T22:12:01+0000] WARNING intento 1/10 de conexión falló: catalogo-bd:27017: [Errno -3] Temporary failure in name resolution (configured timeouts: socketTimeoutMS: 20000.0ms, connectTimeoutMS: 20000.0ms), Timeout: 3.0s, Topology Description: <TopologyDescription id: 6ac8152e579a264c6477e29e, topology_type: Unknown, servers: [<ServerDescription ('catalogo-bd', 27017) server_type: Unknown, rtt: None, error=AutoReconnect('catalogo-bd:27017: [Errno -3] Temporary failure in name resolution (configured timeouts: socketTimeoutMS: 20000.0ms, connectTimeoutMS: 20000.0ms)')>]>
```

### b) Fallo por omisión de `?authSource=admin`

```bash
docker exec -it catalogo-db mongosh "mongodb://catalogo_user:catalogo_pass@localhost:27017/catalogodb" --eval "show dbs"
```

**Mensaje registrado:**

```text
MongoServerError: Authentication failed.
```

### c) Explicación teórica de los dos modos de falla

* **Fallo de resolución de nombre:** Indica que la conexión TCP **no se estableció**. El resolvedor DNS interno de la red de Docker no encontró una entrada IP para el nombre solicitado (`catalogo-bd`), por lo que el paquete no pudo salir hacia la base de datos.

* **Fallo de autenticación:** Indica que el contenedor de la base de datos **sí fue alcanzado exitosamente a nivel de red** y el socket TCP se abrió. El intercambio a nivel de aplicación ocurrió, pero el motor rechazó las credenciales al intentar validarlas contra el catálogo local de `catalogodb` en lugar de la base `admin`.

---

## 5. Análisis del Upstream de Nginx sin Red

Comando ejecutado sin parámetro `--network`:

```bash
docker run -d \
  --name catalogo-frontend \
  -e API_HOST=catalogo-api \
  -e API_PORT=8000 \
  -p 3000:8080 \
  catalogo-frontend:v1
```

### Salida de `docker ps -a`

```text
CONTAINER ID   IMAGE                  COMMAND                  CREATED          STATUS                      PORTS    NAMES
bf10c9ced646   catalogo-frontend:v1   "/docker-entrypoint.…"   12 seconds ago   Exited (1) 12 seconds ago    catalogo-frontend
```

### Salida textual de `docker logs catalogo-frontend`

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
2026/10/09 13:41:05 [emerg] 1#1: host not found in upstream "catalogo-api" in /etc/nginx/conf.d/default.conf:24
nginx: [emerg] host not found in upstream "catalogo-api" in /etc/nginx/conf.d/default.conf:24
```

### Explicación técnica

El frontend se ejecutó sin conectarlo a la red `catalogo-net`. Al iniciar, Nginx procesó la plantilla de configuración con `envsubst`, pero no pudo resolver el nombre DNS `catalogo-api` porque ambos contenedores no compartían una red Docker adecuada. Como Nginx necesita resolver ese upstream al cargar la configuración, el proceso falló y el contenedor terminó con código de salida `1`.

---

## 6. Integración Completa y Tráfico de Red

Comando con conexión a `catalogo-net`:

```bash
docker rm -f catalogo-frontend
docker run -d \
  --name catalogo-frontend \
  --network catalogo-net \
  -e API_HOST=catalogo-api \
  -e API_PORT=8000 \
  -p 3000:8080 \
  catalogo-frontend:v1
```

### Observaciones de la pestaña Network en DevTools

* **Ruta de la solicitud:** La petición HTTP POST/GET se despacha hacia el mismo origen: `http://localhost:3000/api/productos`.

* **Ausencia de peticiones OPTIONS (CORS):** Al operar sobre un origen unificado servido por el reverse proxy de Nginx (puerto 3000), el navegador interpreta la comunicación como *same-origin*. No se disparan solicitudes de pre-vuelo HTTP `OPTIONS` ni se requieren cabeceras CORS en la API.

---

## 7. Verificación del Proxy y Prefijo `ROOT_PATH`

### a) Comparación de respuestas vía API directa y vía Proxy

* Directo a la API:

```bash
curl -s http://localhost:8000/productos
```

* A través del reverse proxy:

```bash
curl -s http://localhost:3000/api/productos
```

**Resultado:** Ambas peticiones devuelven idéntica estructura y contenido JSON de productos.

### b) Inspección de `openapi.json` con `ROOT_PATH=/api`

```bash
curl -s http://localhost:8000/openapi.json | grep "servers"
```

**Salida:**

```json
"servers":[{"url":"/api"}]
```

### c) Inspección de `openapi.json` con `ROOT_PATH` vacío

```text
servers: (clave ausente)
```

### d) Diferencia observada en `http://localhost:3000/api/docs`

* **Con `ROOT_PATH` vacío:** La interfaz interactiva de Swagger UI asume que la raíz es `/` e intenta realizar las consultas hacia `/productos` (en lugar de `/api/productos`). Estas peticiones devuelven el archivo `index.html` del frontend con código 200 o un error 404, haciendo que la documentación falle o quede en blanco.

* **Con `ROOT_PATH=/api`:** FastAPI inyecta la URL base `/api` en el esquema OpenAPI (`servers: [{"url": "/api"}]`), permitiendo que el cliente Swagger dirija todas las operaciones interactivas bajo la ruta prefijada del proxy.

---

## 8. Persistencia de Datos y Modelo Stateless

Comandos ejecutados:

```bash
# Destrucción del contenedor de base
docker rm -f catalogo-db

# Comprobación de persistencia del volumen
docker volume ls | grep catalogo-db-data

# Recreación del motor con el mismo volumen
docker run -d \
  --name catalogo-db \
  --network catalogo-net \
  -v catalogo-db-data:/data/db \
  -e MONGO_INITDB_ROOT_USERNAME=catalogo_user \
  -e MONGO_INITDB_ROOT_PASSWORD=catalogo_pass \
  mongo:7

# Reinicio del backend para restablecer el pool de conexiones
docker restart catalogo-api
```

### Conteo y verificación posterior del producto con imagen

```bash
curl -s http://localhost:8000/productos
```

**Salida:**

```json
[{"sku":546,"nombre":"56456456","precio":0.01,"categoria":"1","stock":1,"descripcion":"465456321","imagen_tipo":"image/jpeg","imagen_hash":"c9e1c99c01d2d401","creado_en":"2026-10-09T13:50:49.985000","id":"6ac8f139f5d04e427166be06","imagen":true},{"sku":999666,"categoria":"Apple","creado_en":"2026-10-08T21:14:34.858000","descripcion":"Clásico de 4.7\" con Touch ID y cuerpo de aluminio.","imagen_hash":"7e23251b780a76a1","imagen_tipo":"image/jpeg","nombre":"Apple iPhone 6 (Gold, 32 GB)","precio":44000,"stock":2,"id":"6ac807ba828306d4e562e515","imagen":true},{"sku":379182,"categoria":"Apple","creado_en":"2026-10-08T21:14:34.860000","descripcion":"Pantalla Retina HD de 5.5\", carga inalámbrica y chip A11.","imagen_hash":"27cf5a0cc72a5c03","imagen_tipo":"image/jpeg","nombre":"Apple iPhone 8 Plus (Silver, 64 GB)","precio":48000,"stock":6,"id":"6ac807ba828306d4e562e51b","imagen":true},{"sku":456654,"categoria":"Apple","creado_en":"2026-10-08T21:14:34.854000","descripcion":"Super Retina de 5.8\", FaceID y chip A11 Bionic.","imagen_hash":"8101c6ff7c22f74d","imagen_tipo":"image/jpeg","nombre":"Apple iPhone X (Space Gray, 256 GB)","precio":25000,"stock":3,"id":"6ac807ba828306d4e562e50b","imagen":true},{"sku":963852,"categoria":"Asus","creado_en":"2026-10-08T21:14:34.857000","descripcion":"Batería de 4000 mAh que funciona también como power bank.","imagen_hash":"95255abbb146483b","imagen_tipo":"image/jpeg","nombre":"Asus ZenFone Max M1 (Black, 32 GB) (3 GB RAM)","precio":84000,"stock":0,"id":"6ac807ba828306d4e562e513","imagen":true},{"sku":123222,"categoria":"Honor","creado_en":"2026-10-08T21:14:34.855000","descripcion":"Pantalla FullView de 5.84\" con notch y cámara dual de 13 MP.","imagen_hash":"1e45d7360052e1e1","imagen_tipo":"image/jpeg","nombre":"Honor 9N (Midnight Black, 32 GB) (3 GB RAM)","precio":39000,"stock":11,"id":"6ac807ba828306d4e562e50f","imagen":true},{"sku":741852,"categoria":"Lenovo","creado_en":"2026-10-08T21:14:34.856000","descripcion":"Procesador Helio X23 de diez núcleosy Android puro.","imagen_hash":"5bf64a1fb53294cd","imagen_tipo":"image/jpeg","nombre":"Lenovo K8 Note (Venom Black, 64 GB) (4 GB RAM)","precio":36000,"stock":5,"id":"6ac807ba828306d4e562e511","imagen":true},{"sku":357842,"categoria":"OPPO","creado_en":"2026-10-08T21:14:34.853000","descripcion":"Lector de huellas bajo la pantalla AMOLED y cámara frontal de 25 MP.","imagen_hash":"24b3652bc891267c","imagen_tipo":"image/jpeg","nombre":"OPPO K1 (Piano Black, 64 GB) (4 GB RAM)","precio":65000,"stock":6,"id":"6ac807ba828306d4e562e509","imagen":true},{"sku":852369,"categoria":"Xiaomi","creado_en":"2026-10-08T21:14:34.851000","descripcion":"Pantalla 5.99\" HD+, cámara dual de 12 MP y batería de 3080 mAh.","imagen_hash":"3926b6235eecc7b3","imagen_tipo":"image/jpeg","nombre":"Redmi Y2 (Black, 32 GB) (3 GB RAM)","precio":25000,"stock":14,"id":"6ac807ba828306d4e562e505","imagen":true},{"sku":848484,"categoria":"Samsung","creado_en":"2026-10-08T21:14:34.859000","descripcion":"Super AMOLED de 6.3\" y cámaradual de 16 MP + 24 MP.","imagen_hash":"6e0b6f7b0f8514c0","imagen_tipo":"image/jpeg","nombre":"Samsung Galaxy A8 Star (White, 64 GB) (6 GB RAM)","precio":78000,"stock":8,"id":"6ac807ba828306d4e562e519","imagen":true},{"sku":332211,"categoria":"Samsung","creado_en":"2026-10-08T21:14:34.858000","descripcion":"Primer smartphone con cuatro cámaras traseras y pantalla de 6.3\".","imagen_hash":"97b0b86fea2f5dc7","imagen_tipo":"image/jpeg","nombre":"Samsung Galaxy A9 (Bubblegum Pink, 128 GB)","precio":97000,"stock":4,"id":"6ac807ba828306d4e562e517","imagen":true},{"sku":159753,"categoria":"Samsung","creado_en":"2026-10-08T21:14:34.852000","descripcion":"Batería de 5000 mAh con carga rápida y pantalla Infinity-V de 6.3\".","imagen_hash":"178a6127bb70f3cb","imagen_tipo":"image/jpeg","nombre":"Samsung Galaxy M20 (Charcoal Black, 4+64GB)","precio":58000,"stock":9,"id":"6ac807ba828306d4e562e507","imagen":true},{"sku":358426,"categoria":"Vivo","creado_en":"2026-10-08T21:14:34.854000","descripcion":"Cámara frontal pop-up de 32 MP y triple cámara trasera de 48 MP.","imagen_hash":"11814686e87c1818","imagen_tipo":"image/jpeg","nombre":"Vivo V15 Pro (Topaz Blue, 128 GB) (6 GB RAM)","precio":55000,"stock":7,"id":"6ac807ba828306d4e562e50d","imagen":true}]
```

### Explicación del comportamiento

El contenedor de la API opera de forma completamente *stateless* (sin estado local): no guarda archivos multimedia en el sistema de archivos del contenedor. Los bytes de las imágenes y los documentos se alojan directamente en la base MongoDB, la cual almacena su estado en el volumen persistente `catalogo-db-data` montado en `/data/db`. Al recrear el contenedor de MongoDB, los bloques del volumen se vinculan nuevamente sin pérdida de información.

---

## 9. Orden de Arranque y Dependencias

Documentación incorporada al `README.md` del proyecto:

```markdown
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

> **Limitación:** El orden actual reside exclusivamente en la coordinación manual del operador, careciendo de automatización declarativa para la gestión de dependencias y tiempos de espera.
```
