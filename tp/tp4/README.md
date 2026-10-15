# Informe - Trabajo Práctico N° 4

**Asignatura:** Contenedores, Docker y Orquestación con Kubernetes
**Variante:** C - catalogo-app [React + Vite, Python + FastAPI, MongoDB 7]
**Alumno:** César Ezequiel Herrera

---

## 1. Declaración de la Infraestructura con Docker Compose

### Eliminación de la red y contenedores de la semana anterior

Comando ejecutado para evitar conflictos de etiquetas (*labels*) en Docker Compose:

```bash
docker rm -f catalogo-db catalogo-api catalogo-frontend
docker network rm catalogo-net
```

### Contrato de variables de entorno (`.env.example`)

```sh
# ==========================================
# Base de Datos (Mongo:7) - Consumido por catalogo-db
# ==========================================
MONGO_INITDB_ROOT_USERNAME=catalogo_user
MONGO_INITDB_ROOT_PASSWORD=catalogo_pass

# ==========================================
# Backend (FastAPI) - Consumido por catalogo-api
# ==========================================
DB_HOST=catalogo-db
DB_PORT=27017
DB_NAME=catalogodb
DB_USER=catalogo_user
DB_PASSWORD=catalogo_pass
APP_VERSION=v1
ROOT_PATH=/api
SEED_DEMO=true

# ==========================================
# Frontend (Nginx/React) - Consumido por catalogo-frontend
# ==========================================
API_HOST=catalogo-api
API_PORT=8000
```

*Verificación en `.gitignore`:* `.env` se encuentra excluido de Git para prevenir la fuga de credenciales al repositorio remoto.

### Archivo `docker-compose.yml` completo

```yaml
services:
  catalogo-db:
    image: mongo:7
    container_name: catalogo-db
    environment:
      MONGO_INITDB_ROOT_USERNAME: catalogo_user
      MONGO_INITDB_ROOT_PASSWORD: catalogo_pass
    volumes:
      - catalogo-db-data:/data/db
    networks:
      - catalogo-net
    healthcheck:
      test: ["CMD-SHELL", "mongosh --quiet --eval \"db.adminCommand('ping').ok\" || exit 1"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
    cap_drop:
      - ALL
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - SETUID
      - SETGID

  catalogo-api:
    build: ./backend
    image: catalogo-api:v1
    container_name: catalogo-api
    environment:
      DB_HOST: catalogo-db
      DB_PORT: 27017
      DB_NAME: catalogodb
      DB_USER: catalogo_user
      DB_PASSWORD: catalogo_pass
      APP_VERSION: v1
      ROOT_PATH: /api
      SEED_DEMO: "true"
    ports:
      - "8000:8000"
    networks:
      - catalogo-net
    depends_on:
      catalogo-db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('[http://127.0.0.1:8000/health](http://127.0.0.1:8000/health)')"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
    cap_drop:
      - ALL

  catalogo-frontend:
    build: ./frontend
    image: catalogo-frontend:v1
    container_name: catalogo-frontend
    environment:
      API_HOST: catalogo-api
      API_PORT: 8000
    ports:
      - "3000:8080"
    networks:
      - catalogo-net
    depends_on:
      catalogo-api:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "[http://127.0.0.1:8080/healthz](http://127.0.0.1:8080/healthz)"]
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s
    cap_drop:
      - ALL

networks:
  catalogo-net:
    name: catalogo-net

volumes:
  catalogo-db-data:
    name: catalogo-db-data
```

---

## 2. Healthcheck de `catalogo-db`

* **Comando:** `mongosh --quiet --eval "db.adminCommand('ping').ok" || exit 1`

* **Análisis de seguridad y ventajas:** La prueba `ping` no requiere autenticación en el motor MongoDB. Esto constituye una ventaja respecto a otros motores de base de datos porque evita exponer la contraseña en texto plano en la definición del contenedor o en la salida de `docker inspect`.

* **Alcance de la comprobación (Qué no verifica):** Un resultado exitoso en la orden `ping` confirma únicamente que el proceso `mongod` está activo y respondiendo solicitudes básicas a nivel de red/socket en el puerto 27017. No valida que la base de datos de la aplicación esté disponible, ni que el usuario `catalogo_user` exista o que la contraseña configurada sea la correcta.

## 3. Healthcheck de `catalogo-api`

* **Comando:** `python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')"`

* **Justificación técnica:** Se utiliza la librería estándar `urllib.request` de Python para evitar instalar clientes adicionales como `curl` o `wget` dentro de la imagen final runtime. Esto respeta la práctica de reducir la superficie de ataque y mantener el tamaño de la imagen al mínimo sin agregar dependencias innecesarias.

## 4. Healthcheck de `catalogo-frontend`

* **Comando:** `wget --no-verbose --tries=1 --spider http://127.0.0.1:8080/healthz`

* **Justificación técnica:** Utiliza la utilidad `wget` provista de forma nativa por BusyBox en Alpine Linux, realizando una consulta de verificación de presencia HTTP (*spider*) a la ruta del servidor web de Nginx sin descargar el contenido.

---

## 5. Coordinación de Arranque (`depends_on` con `service_healthy`)

### a) `catalogo-api` respecto de `catalogo-db`

La API necesita que MongoDB no solo haya iniciado su proceso, sino que se encuentre en estado *healthy* aceptando conexiones socket activas. Si la API intentara arrancar antes de que la base complete su proceso de inicio, fallaría al intentar conectarse y sembrar la base de datos. La sintaxis `depends_on` simple sólo aguarda el inicio del contenedor, mientras que `condition: service_healthy` posterga el arranque hasta superar las pruebas de salud.

### b) `catalogo-frontend` respecto de `catalogo-api`

Nginx evalúa los nombres de host declarados en su directiva `proxy_pass` al momento de arrancar. Si `catalogo-api` no existiera o no tuviera una dirección IP asignada en la red `catalogo-net`, Nginx fallaría inmediatamente con `host not found in upstream`. El uso de `service_healthy` asegura además que el backend se encuentre listo para responder solicitudes, evitando errores HTTP 502 Bad Gateway en las primeras peticiones del usuario.

---

## 4. Despliegue y Verificación de Estado

### a) Validación de sintaxis (`docker compose config`)

```yaml
name: catalogo-app
services:
  catalogo-api:
    build:
      context: /home/estudiante/catalogo-app/backend
      dockerfile: Dockerfile
    cap_drop:
      - ALL
    container_name: catalogo-api
    depends_on:
      catalogo-db:
        condition: service_healthy
        required: true
    environment:
      APP_VERSION: v1
      DB_HOST: catalogo-db
      DB_NAME: catalogodb
      DB_PASSWORD: catalogo_pass
      DB_PORT: "27017"
      DB_USER: catalogo_user
      ROOT_PATH: /api
      SEED_DEMO: "true"
    healthcheck:
      test:
        - CMD
        - python
        - -c
        - import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')
      timeout: 3s
      interval: 5s
      retries: 10
      start_period: 10s
    image: catalogo-api:v1
    networks:
      catalogo-net: null
    ports:
      - mode: ingress
        target: 8000
        published: "8000"
        protocol: tcp
  catalogo-db:
    cap_add:
      - CHOWN
      - DAC_OVERRIDE
      - FOWNER
      - SETUID
      - SETGID
    cap_drop:
      - ALL
    container_name: catalogo-db
    environment:
      MONGO_INITDB_ROOT_PASSWORD: catalogo_pass
      MONGO_INITDB_ROOT_USERNAME: catalogo_user
    healthcheck:
      test:
        - CMD-SHELL
        - mongosh --quiet --eval "db.adminCommand('ping').ok" || exit 1
      timeout: 3s
      interval: 5s
      retries: 10
      start_period: 10s
    image: mongo:7
    networks:
      catalogo-net: null
    volumes:
      - type: volume
        source: catalogo-db-data
        target: /data/db
        volume: {}
  catalogo-frontend:
    build:
      context: /home/estudiante/catalogo-app/frontend
      dockerfile: Dockerfile
    cap_drop:
      - ALL
    container_name: catalogo-frontend
    depends_on:
      catalogo-api:
        condition: service_healthy
        required: true
    environment:
      API_HOST: catalogo-api
      API_PORT: "8000"
    healthcheck:
      test:
        - CMD
        - wget
        - --no-verbose
        - --tries=1
        - --spider
        - http://127.0.0.1:8080/healthz
      timeout: 3s
      interval: 5s
      retries: 5
      start_period: 5s
    image: catalogo-frontend:v1
    networks:
      catalogo-net: null
    ports:
      - mode: ingress
        target: 8080
        published: "3000"
        protocol: tcp
networks:
  catalogo-net:
    name: catalogo-net
volumes:
  catalogo-db-data:
    name: catalogo-db-data
```

### b) Construcción y despliegue (`docker compose up --build -d`)

```text
[+] Building 1.5s (37/37) FINISHED
 => [internal] load local bake definitions                                                                                                  0.0s
 => => reading from stdin 1.03kB                                                                                                            0.0s
 => [catalogo-frontend internal] load build definition from Dockerfile                                                                      0.0s
 => => transferring dockerfile: 527B                                                                                                        0.0s
 => [catalogo-api internal] load build definition from Dockerfile                                                                           0.0s
 => => transferring dockerfile: 594B                                                                                                        0.0s
 => [catalogo-api internal] load metadata for docker.io/library/python:3.12-slim                                                            1.4s
 => [catalogo-api internal] load metadata for docker.io/library/python:3.12                                                                 1.3s
 => [catalogo-frontend internal] load metadata for docker.io/nginxinc/nginx-unprivileged:1.27-alpine                                        1.2s
 => [catalogo-frontend internal] load metadata for docker.io/library/node:20-alpine                                                         1.1s
 => [auth] library/node:pull token for registry-1.docker.io                                                                                 0.0s
 => [auth] library/python:pull token for registry-1.docker.io                                                                               0.0s
 => [auth] nginxinc/nginx-unprivileged:pull token for registry-1.docker.io                                                                  0.0s
 => [catalogo-frontend internal] load .dockerignore                                                                                         0.0s
 => => transferring context: 2B                                                                                                             0.0s
 => [catalogo-frontend builder 1/6] FROM docker.io/library/node:20-alpine@sha256:fb4cd12c85ee03686f6af5362a0b0d56d50c58a04632e6c0fb8363f60  0.0s
 => => resolve docker.io/library/node:20-alpine@sha256:fb4cd12c85ee03686f6af5362a0b0d56d50c58a04632e6c0fb8363f609372293                     0.0s
 => [catalogo-frontend stage-1 1/4] FROM docker.io/nginxinc/nginx-unprivileged:1.27-alpine@sha256:65e3e85dbaed8ba248841d9d58a899b6197106c2  0.0s
 => => resolve docker.io/nginxinc/nginx-unprivileged:1.27-alpine@sha256:65e3e85dbaed8ba248841d9d58a899b6197106c23cb0ff1a132b7bfe0547e4c0    0.0s
 => [catalogo-frontend internal] load build context                                                                                         0.0s
 => => transferring context: 770B                                                                                                           0.0s
 => CACHED [catalogo-frontend stage-1 2/4] WORKDIR /usr/share/nginx/html                                                                    0.0s
 => CACHED [catalogo-frontend builder 2/6] WORKDIR /app                                                                                     0.0s
 => CACHED [catalogo-frontend builder 3/6] COPY package*.json ./                                                                            0.0s
 => CACHED [catalogo-frontend builder 4/6] RUN npm ci                                                                                       0.0s
 => CACHED [catalogo-frontend builder 5/6] COPY . .                                                                                         0.0s
 => CACHED [catalogo-frontend builder 6/6] RUN npm run build                                                                                0.0s
 => CACHED [catalogo-frontend stage-1 3/4] COPY --from=builder /app/dist .                                                                  0.0s
 => CACHED [catalogo-frontend stage-1 4/4] COPY nginx/templates/default.conf.template /etc/nginx/templates/default.conf.template            0.0s
 => [catalogo-frontend] exporting to image                                                                                                  0.0s
 => => exporting layers                                                                                                                     0.0s
 => => exporting manifest sha256:62ce28af35eeb71eae501a8ab2ca2fa704729a22022b8c436e962fdfd29cd63a                                           0.0s
 => => exporting config sha256:03e88f649122b8b671afb867dea61d376b9416033852cb40d202e42508370a59                                             0.0s
 => => exporting attestation manifest sha256:c3a19cdb432d00d400f8808af42fafc3f70a9bcd88d68cd64a0e3e6774a14c34                               0.0s
 => => exporting manifest list sha256:8fc0a96695dd44c7632b12f3caf9b408506cb5d8d453ce1dc5320db249f1cba1                                      0.0s
 => => naming to docker.io/library/catalogo-frontend:v1                                                                                     0.0s
 => => unpacking to docker.io/library/catalogo-frontend:v1                                                                                  0.0s
 => [catalogo-frontend] resolving provenance for metadata file                                                                              0.0s
 => [catalogo-api internal] load .dockerignore                                                                                              0.0s
 => => transferring context: 74B                                                                                                            0.0s
 => [catalogo-api builder 1/4] FROM docker.io/library/python:3.12@sha256:42dc0ee40d7707e2f2ee506a1cbe52e09ab11c207c5b429342638f83177601fe   0.0s
 => => resolve docker.io/library/python:3.12@sha256:42dc0ee40d7707e2f2ee506a1cbe52e09ab11c207c5b429342638f83177601fe                        0.0s
 => [catalogo-api stage-1 1/5] FROM docker.io/library/python:3.12-slim@sha256:a6e34c598f2467ed0e9a8d349809fcd8b5c603269512df273a0bb1784edc  0.0s
 => => resolve docker.io/library/python:3.12-slim@sha256:a6e34c598f2467ed0e9a8d349809fcd8b5c603269512df273a0bb1784edc11b1                   0.0s
 => [catalogo-api internal] load build context                                                                                              0.0s
 => => transferring context: 683B                                                                                                           0.0s
 => CACHED [catalogo-api stage-1 2/5] WORKDIR /app                                                                                          0.0s
 => CACHED [catalogo-api stage-1 3/5] RUN useradd -m -u 1000 appuser                                                                        0.0s
 => CACHED [catalogo-api builder 2/4] WORKDIR /app                                                                                          0.0s
 => CACHED [catalogo-api builder 3/4] COPY requirements.txt .                                                                               0.0s
 => CACHED [catalogo-api builder 4/4] RUN pip install --user --no-cache-dir -r requirements.txt                                             0.0s
 => CACHED [catalogo-api stage-1 4/5] COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local                         0.0s
 => CACHED [catalogo-api stage-1 5/5] COPY --chown=appuser:appuser . .                                                                      0.0s
 => [catalogo-api] exporting to image                                                                                                       0.0s
 => => exporting layers                                                                                                                     0.0s
 => => exporting manifest sha256:5eb52fca46c6be82d0bbe13727b6686045870ff7eb8dcaf636a400f03b670868                                           0.0s
 => => exporting config sha256:1f0f02b37e3652fa4db60de3b843356365bc968af09c30ad571669fc91cba1be                                             0.0s
 => => exporting attestation manifest sha256:c9f29d97796f512d35ba77b0b0c56d35ff86d1b1775e331888e180079be294bf                               0.0s
 => => exporting manifest list sha256:ec9b330ce8544c9a073799efbf3814599a507c246801cb253abb030cbfd5af81                                      0.0s
 => => naming to docker.io/library/catalogo-api:v1                                                                                          0.0s
 => => unpacking to docker.io/library/catalogo-api:v1                                                                                       0.0s
 => [catalogo-api] resolving provenance for metadata file                                                                                   0.0s
WARN[0001] volume "catalogo-db-data" already exists but was not created by Docker Compose. Use `external: true` to use an existing volume
[+] up 6/6
 ✔ Image catalogo-frontend:v1  Built                                                                                                         1.6s
 ✔ Image catalogo-api:v1       Built                                                                                                         1.6s
 ✔ Network catalogo-net        Created                                                                                                       0.0s
 ✔ Container catalogo-db       Healthy                                                                                                       6.2s
 ✔ Container catalogo-api      Healthy                                                                                                      12.3s
 ✔ Container catalogo-frontend Started                                                                                                      12.4s
```

### c) Estado de los servicios (`docker compose ps`)

```text
NAME                IMAGE                  COMMAND                  SERVICE             CREATED          STATUS                    PORTS
catalogo-api        catalogo-api:v1        "uvicorn main:app --…"   catalogo-api        19 seconds ago   Up 13 seconds (healthy)   0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp
catalogo-db         mongo:7                "docker-entrypoint.s…"   catalogo-db         19 seconds ago   Up 19 seconds (healthy)   27017/tcp
catalogo-frontend   catalogo-frontend:v1   "/docker-entrypoint.…"   catalogo-frontend   19 seconds ago   Up 7 seconds (healthy)    0.0.0.0:3000->8080/tcp, [::]:3000->8080/tcp
```

### d) Registros del backend (`docker compose logs catalogo-api`)

```text
catalogo-api  | catalogo-api v1 arrancando (base=catalogodb)
catalogo-api  | INFO:     Started server process [1]
catalogo-api  | INFO:     Waiting for application startup.
catalogo-api  | [2026-10-13T20:09:30+0000] INFO conexión a MongoDB establecida (base=catalogodb)
catalogo-api  | [2026-10-13T20:09:30+0000] INFO catálogo de demo sembrado (12 productos)
catalogo-api  | INFO:     Application startup complete.
catalogo-api  | INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
catalogo-api  | INFO:     127.0.0.1:55796 - "GET /health HTTP/1.1" 200 OK
catalogo-api  | INFO:     127.0.0.1:55808 - "GET /health HTTP/1.1" 200 OK
catalogo-api  | INFO:     127.0.0.1:40088 - "GET /health HTTP/1.1" 200 OK
catalogo-api  | INFO:     127.0.0.1:52006 - "GET /health HTTP/1.1" 200 OK
```

---

## 5. Ciclo de Vida del Dato y Persistencia

### Comprobación de reinicio con preservación de volumen (`docker compose down`)

```bash
docker compose down
docker compose up -d
curl -s http://localhost:3000/api/productos | jq -c 'map({sku, nombre})'
```

**Resultado:**

```json
[{"sku":546,"nombre":"56456456"},{"sku":999666,"nombre":"Apple iPhone 6 (Gold, 32 GB)"},{"sku":379182,"nombre":"Apple iPhone 8 Plus (Silver, 64GB)"},{"sku":456654,"nombre":"Apple iPhone X (Space Gray, 256 GB)"},{"sku":963852,"nombre":"Asus ZenFone Max M1 (Black, 32 GB) (3 GB RAM)"},{"sku":123222,"nombre":"Honor 9N (Midnight Black, 32 GB) (3 GB RAM)"},{"sku":741852,"nombre":"Lenovo K8 Note (Venom Black, 64 GB) (4 GB RAM)"},{"sku":357842,"nombre":"OPPO K1 (Piano Black, 64 GB) (4 GB RAM)"},{"sku":852369,"nombre":"Redmi Y2 (Black, 32 GB) (3 GB RAM)"},{"sku":848484,"nombre":"Samsung Galaxy A8 Star (White, 64 GB) (6 GB RAM)"},{"sku":332211,"nombre":"Samsung Galaxy A9 (Bubblegum Pink, 128 GB)"},{"sku":159753,"nombre":"Samsung Galaxy M20 (Charcoal Black, 4+64GB)"},{"sku":358426,"nombre":"Vivo V15 Pro (Topaz Blue, 128 GB) (6 GB RAM)"}]
```

### Comprobación de eliminación con destrucción de volumen (`docker compose down -v`)

```bash
docker compose down -v
docker compose up -d
curl -s http://localhost:3000/api/productos | jq -c 'map({sku, nombre})'
```

**Resultado:**

```text
[+] down 5/5
 ✔ Container catalogo-frontend Removed                                  0.6s
 ✔ Container catalogo-api      Removed                                  0.6s
 ✔ Container catalogo-db       Removed                                  0.1s
 ✔ Network catalogo-net        Removed                                  0.1s
 ✔ Volume catalogo-db-data     Removed                                  0.0s
[+] up 5/5
 ✔ Network catalogo-net        Created                                  0.0s
 ✔ Volume catalogo-db-data     Created                                  0.0s
 ✔ Container catalogo-db       Healthy                                  5.7s
 ✔ Container catalogo-api      Healthy                                 11.2s
 ✔ Container catalogo-frontend Started                                 11.4s
```

```json
[{"sku":999666,"nombre":"Apple iPhone 6 (Gold, 32 GB)"},{"sku":379182,"nombre":"Apple iPhone 8 Plus (Silver, 64 GB)"},{"sku":456654,"nombre":"Apple iPhone X (Space Gray, 256 GB)"},{"sku":963852,"nombre":"Asus ZenFone Max M1 (Black, 32 GB) (3 GB RAM)"},{"sku":123222,"nombre":"Honor 9N (Midnight Black, 32 GB) (3 GB RAM)"},{"sku":741852,"nombre":"Lenovo K8 Note (Venom Black, 64 GB) (4 GB RAM)"},{"sku":357842,"nombre":"OPPO K1 (Piano Black, 64 GB) (4 GB RAM)"},{"sku":852369,"nombre":"Redmi Y2 (Black, 32 GB) (3 GB RAM)"},{"sku":848484,"nombre":"Samsung Galaxy A8 Star (White, 64 GB) (6 GB RAM)"},{"sku":332211,"nombre":"Samsung Galaxy A9 (Bubblegum Pink, 128 GB)"},{"sku":159753,"nombre":"Samsung Galaxy M20 (Charcoal Black, 4+64GB)"},{"sku":358426,"nombre":"Vivo V15 Pro (Topaz Blue, 128 GB) (6 GB RAM)"}]
```

`docker compose down` remueve únicamente los contenedores y la red virtual, preservando los volúmenes nombrados intactos en el anfitrión. En cambio, `docker compose down -v` elimina de forma irreversible los volúmenes de datos declarados en el proyecto, destruyendo la persistencia de MongoDB.

---

## 6. Verificación de Idempotencia en la Siembra de Datos

Comandos ejecutados:

```bash
# Consultar el conteo de productos en el catálogo
curl -s http://localhost:3000/api/productos | grep -o '"sku"' | wc -l

# Forzar la recreación del contenedor de la API
docker compose up -d --force-recreate catalogo-api

# Consultar el conteo de productos en el catálogo después de recrear
curl -s http://localhost:3000/api/productos | grep -o '"sku"' | wc -l
```

**Resultado obtenido:**

```text
12
```

```text
[+] up 2/2
 ✔ Container catalogo-db  Healthy                                     1.3s
 ✔ Container catalogo-api Started                                     1.4s
```

```text
12
```

La siembra de datos implementa una operación de tipo *upsert* basada en el nombre de los productos. Al reiniciar o escalar la API en múltiples réplicas simultáneas, la ejecución recurrente de la rutina de siembra actualiza las entidades existentes sin generar duplicados en la colección.

---

## 7. Auditoría de Seguridad y Mínimo Privilegio

### a) Verificación de identidades de usuario runtime

```bash
docker compose exec catalogo-api id
docker compose exec catalogo-frontend id
```

**Salida obtenida:**

```text
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
```

```text
uid=101(nginx) gid=101(nginx) groups=101(nginx)
```

Ambas imágenes personalizadas ejecutan sus procesos bajo usuarios sin privilegios (UID distinto de 0).

### b) Mapeo y exposición de puertos (`docker compose ps`)

* `catalogo-db`: Ningún puerto publicado hacia el anfitrión (acceso restrictivo interno a `catalogo-net`).

* `catalogo-api`: Puerto `8000:8000` publicado únicamente para propósitos de depuración en fase local (pasará a ser exclusivo interno en etapas posteriores).

* `catalogo-frontend`: Puerto `3000:8080` publicado como único punto de acceso expuesto para los clientes web.

### c) Escaneo de vulnerabilidades con Trivy

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image catalogo-api:v1
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image catalogo-frontend:v1
```

**Tabla resumida de auditoría de vulnerabilidades:**

| Imagen | Total Vulnerabilidades | Sistema Base (OS) | Dependencias de Lenguaje | Con Versión Corregida (Fixed) |
| --- | --- | --- | --- | --- |
| `catalogo-api:v1` | 183 | 166 | 17 | 18 |
| `catalogo-frontend:v1` | 159 | 159 | 0 | 159 |

---

## 8. Principio de Mínimo Privilegio y Capabilities (`cap_drop` / `cap_add`)

### Remoción global de capacidades (`cap_drop: [ALL]`)

Al aplicar `cap_drop: [ALL]` a todos los servicios:

* `catalogo-api` y `catalogo-frontend` funcionaron de forma continua e ininterrumpida debido a que corren con usuarios sin privilegios y no apelan a funciones restrictivas del kernel de Linux.

* `catalogo-db` se detuvo inmediatamente al iniciar.

### Registro de fallo en `catalogo-db` sin capabilities

```text
[+] up 4/4
 ✔ Network catalogo-net        Created                                   0.0s
 ✔ Container catalogo-db       Created                                   0.1s
 ✔ Container catalogo-api      Created                                   0.0s
 ✔ Container catalogo-frontend Created                                   0.0s
Attaching to catalogo-api, catalogo-db, catalogo-frontend
catalogo-db  | find: '/data/db/.mongodb': Permission denied
catalogo-db  | find: '/data/db/diagnostic.data': Permission denied
catalogo-db  | find: '/data/db/journal': Permission denied
Container catalogo-db Waiting
catalogo-db exited with code 1
Container catalogo-db Error dependency catalogo-db failed to start
dependency failed to start: container catalogo-db exited (1)
```

### Configuración final de capabilities para `catalogo-db`

Capabilities agregadas gradualmente en `docker-compose.yml`:

```yaml
cap_add:
  - CHOWN
  - DAC_OVERRIDE
  - FOWNER
  - SETUID
  - SETGID
```

### Explicación técnica

El proceso inicial del contenedor oficial `mongo:7` arranca bajo la cuenta `root` para ajustar permisos del directorio montado `/data/db` y configurar el entorno del servidor. Posteriormente, hace uso de las llamadas de sistema `setuid` y `setgid` para degradar su ejecución al usuario sin privilegios `mongodb` (UID 999). Sin las capacidades `SETUID` y `SETGID` concedidas por el kernel, Linux deniega el cambio de identidad y la base de datos aborta con *Operation not permitted*. Por el contrario, la API y el Frontend no inician como root ni cambian de usuario en tiempo de ejecución, por lo que operan de forma segura bajo `cap_drop: [ALL]`.

---

## 9. Comparativa: Docker CLI (`docker run`) vs Docker Compose

| Criterio | Ejecución Manual CLI | Declarativo Docker Compose |
| --- | --- | --- |
| **Definición de Infraestructura** | Múltiples comandos imperativos `docker run` secuenciales y extensos. | Un único manifiesto `docker-compose.yml` versionable en el repositorio. |
| **Aislamiento y Redes** | Creación y vinculación manual con `docker network create` y `--network`. | Creación y gestión automatizada del ciclo de vida de la red. |
| **Coordinación y Tiempos** | Espera manual del operador basada en inspección visual de logs. | Automática mediante `depends_on` con `condition: service_healthy`. |
| **Manejo de Salud (*Healthchecks*)** | Consultas aisladas a mano sin monitoreo dentro del motor de Docker. | Definición declarativa e integrada de sondas de salud en cada contenedor. |
| **Seguridad y Capabilities** | Configuración manual dispersa por comando de terminal. | Restricción estandarizada mediante `cap_drop` y `cap_add` en el archivo de orquestación. |
