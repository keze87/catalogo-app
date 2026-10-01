# Informe - Trabajo Práctico N° 2

**Asignatura:** Contenedores, Docker y Orquestación con Kubernetes  
**Variante:** C - catalogo-app (React + Vite, Python + FastAPI, MongoDB 7)  
**Alumno:** César Herrera

---

## 1. Imagen Ingenua de la API (`catalogo-api:naive`)

### Contenido de `backend/Dockerfile` inicial (una sola etapa)

```dockerfile
FROM python:3.12
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Salida del comando de construcción (`docker build`)

```log
estudiante@curso-docker:~/catalogo-app$ docker build -t catalogo-api:naive ./backend
[+] Building 91.5s (9/9) FINISHED                                                                  docker:default
 => [internal] load build definition from Dockerfile                                                         0.0s
 => => transferring dockerfile: 192B                                                                         0.0s
 => [internal] load metadata for docker.io/library/python:3.12                                               0.3s
 => [internal] load .dockerignore                                                                            0.0s
 => => transferring context: 2B                                                                              0.0s
 => [1/4] FROM docker.io/library/python:3.12@sha256:4d1caded1f729ae443eb803f26ffde7b61e696aeaef62f099abb6d  78.9s
 => => resolve docker.io/library/python:3.12@sha256:4d1caded1f729ae443eb803f26ffde7b61e696aeaef62f099abb6dd  0.0s
 => => sha256:e122006e13eb998fe6ca0fa75154eb43300015ae2181aa27b3684f0120416273 25.74MB / 25.74MB             2.4s
 => => sha256:473e0af857def22304d36a47667d00c3456e915bbff47b10e8d019e96c91dc5e 6.09MB / 6.09MB               2.6s
 => => sha256:38985a14f2b1b8215895ecb448f3dfc4067cb494aa00b547c78c9a012e9b2460 67.81MB / 67.81MB            28.5s
 => => sha256:e7272a918094ebc3b66e08b23223afd37614df4ba4518b2dfd9da5ee9247fce6 236.40MB / 236.40MB          61.1s
 => => sha256:6eefb2f5d3e91a6cfc577476bbec26bb63f0d0fc31f904493f400833783aa2c2 49.38MB / 49.38MB            21.9s
 => => sha256:42f0cc32f2e355552fbfad163210ddc51f7b8bc7cfaddb2a41bd9c4a7c5e3c49 25.64MB / 25.64MB            14.4s
 => => extracting sha256:6eefb2f5d3e91a6cfc577476bbec26bb63f0d0fc31f904493f400833783aa2c2                   11.0s
 => => extracting sha256:42f0cc32f2e355552fbfad163210ddc51f7b8bc7cfaddb2a41bd9c4a7c5e3c49                    3.9s
 => => extracting sha256:38985a14f2b1b8215895ecb448f3dfc4067cb494aa00b547c78c9a012e9b2460                   16.1s
 => => extracting sha256:e7272a918094ebc3b66e08b23223afd37614df4ba4518b2dfd9da5ee9247fce6                   16.4s
 => => extracting sha256:473e0af857def22304d36a47667d00c3456e915bbff47b10e8d019e96c91dc5e                    0.2s
 => => extracting sha256:e122006e13eb998fe6ca0fa75154eb43300015ae2181aa27b3684f0120416273                    1.0s
 => => extracting sha256:6dbb0bf7b6c42adda0467b6d066f6a21dc3359dc53d7fbb45ece6697f965eaa4                    0.0s
 => [internal] load build context                                                                            0.0s
 => => transferring context: 651B                                                                            0.0s
 => [2/4] WORKDIR /app                                                                                       0.2s
 => [3/4] COPY . .                                                                                           0.0s
 => [4/4] RUN pip install -r requirements.txt                                                                8.9s
 => exporting to image                                                                                       2.7s 
 => => exporting layers                                                                                      1.9s 
 => => exporting manifest sha256:5a3d1d827841b8c51f1cf2572947273def4bc2d38eea39e756ba908bc25e360b            0.0s 
 => => exporting config sha256:2be4f66a014bf6f32ef23571f3c21f73f72af7806149f11fdcbd19f28b5b78d6              0.0s 
 => => exporting attestation manifest sha256:cd0755e33ab0180310343e98bf5b86e1d35665241831510aa03781684e94b7  0.0s 
 => => exporting manifest list sha256:e79102cf80bc128db1e3e54f23063c8a49bd96419febc1296459e8da4cbdd30f       0.0s 
 => => naming to docker.io/library/catalogo-api:naive                                                        0.0s
 => => unpacking to docker.io/library/catalogo-api:naive                                                     0.8s
```

### Medición de tamaño (`docker images catalogo-api:naive`)

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
catalogo-api:naive   e79102cf80bc       1.71GB          445MB
```

* **Tamaño obtenido:** 1.71GB

---

## 2. Dockerfile Definitivo Multi-etapa de la API

### Archivo `backend/.dockerignore`

```text
__pycache__
.git
.env
*.pyc
.venv
```

### Contenido de `backend/Dockerfile` optimizado

```dockerfile
# Etapa 1: Builder
FROM python:3.12 AS builder
WORKDIR /app

COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Etapa 2: Runtime
FROM python:3.12-slim
WORKDIR /app

ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION}
ENV PYTHONUNBUFFERED=1

RUN useradd -m -u 1000 appuser

COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local
COPY --chown=appuser:appuser . .

ENV PATH="/home/appuser/.local/bin:${PATH}"

USER appuser

EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 3. Comprobación del Horneado de Versión (`ARG` y `ENV`)

### Construcción de versiones `v1` y `v2`

```bash
docker build -t catalogo-api:v1 ./backend
docker build --build-arg APP_VERSION=v2 -t catalogo-api:v2 ./backend
```

### Verificación de la variable horneada en `v2`

```bash
docker run --rm --entrypoint sh catalogo-api:v2 -c 'echo $APP_VERSION'
```

**Salida obtenida:**

```text
v2
```

* **Explicación técnica:** `ARG` existe exclusivamente durante el proceso de construcción de la imagen (`docker build`). Para que su valor persista y sea accesible en tiempo de ejecución por el contenedor, resulta obligatorio persistirlo en una variable de entorno mediante `ENV APP_VERSION=${APP_VERSION}`.

---

## 4. Imagen Multi-etapa del Frontend (`catalogo-frontend`)

### Contenido de `frontend/Dockerfile`

```dockerfile
# Etapa 1: Builder
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Etapa 2: Servidor web sin privilegios
FROM nginxinc/nginx-unprivileged:1.27-alpine
WORKDIR /usr/share/nginx/html

ENV API_HOST=catalogo-api \
    API_PORT=8000 \
    NGINX_ENVSUBST_FILTER=^API_

COPY --from=builder /app/dist .

EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

### Verificación de propiedades de seguridad y pureza de la imagen

#### a) Verificación de usuario sin privilegios

```bash
docker run --rm catalogo-frontend:v1 whoami
```

**Salida:**

```text
nginx
```

#### b) Ausencia del entorno de Node.js

```bash
docker run --rm catalogo-frontend:v1 node --version
```

**Salida:**

```text
/docker-entrypoint.sh: exec: line 47: node: not found
```

#### c) Tamaño de la imagen del frontend

```bash
docker images catalogo-frontend:v1
```

**Salida:**

```text
IMAGE                  ID             DISK USAGE   CONTENT SIZE   EXTRA
catalogo-frontend:v1   f53ec87915b4       73.9MB           21MB
```

---

## 5. Tabla Comparativa y Medición

Tabla incorporada en la documentación del repositorio (`README.md`):

| Imagen | Tag | Base | Tamaño | Reducción vs Naive | Justificación de la decisión técnica |
| --- | --- | --- | --- | --- | --- |
| `catalogo-api` | `naive` | `python:3.12` | 1.71GB | 0% | Etapa única con sistema base completo y herramientas de compilación innecesarias. |
| `catalogo-api` | `v1` | `python:3.12-slim` | 246MB | 85% | Uso de base `slim`, build multi-etapa e instalación con `--user --no-cache-dir` descartando compiladores del sistema. |
| `catalogo-frontend` | `v1` | `nginxinc/nginx-unprivileged:1.27-alpine` | 73.9MB | N/A | Multi-etapa que desecha Node.js, `npm` y `node_modules`, copiando solo los artefactos estáticos generados en `dist/`. |

---

## 6. Análisis de Capas de la API (`docker history`)

Comando ejecutado:

```bash
docker history catalogo-api:v1
```

**Salida obtenida:**

```text
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
<missing>      53 minutes ago   COPY --chown=appuser:appuser . . # buildkit     553kB     buildkit.dockerfile.v0
<missing>      53 minutes ago   COPY --chown=appuser:appuser /root/.local /h…   53MB      buildkit.dockerfile.v0
<missing>      53 minutes ago   RUN |1 APP_VERSION=v1 /bin/sh -c useradd -m …   69.6kB    buildkit.dockerfile.v0
<missing>      53 minutes ago   WORKDIR /app                                    8.19kB    buildkit.dockerfile.v0
<missing>      12 days ago      RUN /bin/sh -c set -eux;  for src in idle3 p…   16.4kB    buildkit.dockerfile.v0
<missing>      12 days ago      RUN /bin/sh -c set -eux;   savedAptMark="$(a…   41.4MB    buildkit.dockerfile.v0
<missing>      12 days ago      RUN /bin/sh -c set -eux;  apt-get update;  a…   4.94MB    buildkit.dockerfile.v0
<missing>      13 days ago      # debian.sh --arch 'amd64' out/ 'trixie' '@1…   87.6MB    debuerreotype 0.17
```

### Identificación de la capa principal

* **Capa de mayor tamaño absoluto:** Corresponde a la imagen base del sistema operativo (`python:3.12-slim`, construida sobre Debian GNU/Linux), la cual aporta el sistema de archivos base y las dependencias de sistema mínimas para ejecutar Linux.
* **Capa de mayor tamaño generada en nuestro build:** Corresponde a la instrucción `COPY --chown=appuser:appuser /root/.local /home/appuser/.local`. Representa el conjunto de paquetes Python instalados por `pip` durante la etapa constructora (`builder`), tales como FastAPI, Uvicorn, Pydantic, motor de MongoDB y sus dependencias compiladas.

---

## 7. Efecto del Cache de Construcción

### a) Modificación de código fuente (`backend/*.py`)

Salida parcial de la reconstrucción:

```log
[+] Building 0.5s (15/15) FINISHED                                                                 docker:default
 => [internal] load build definition from Dockerfile                                                         0.0s
 => => transferring dockerfile: 594B                                                                         0.0s
 => [internal] load metadata for docker.io/library/python:3.12-slim                                          0.3s
 => [internal] load metadata for docker.io/library/python:3.12                                               0.3s
 => [internal] load .dockerignore                                                                            0.0s
 => => transferring context: 74B                                                                             0.0s
 => [builder 1/4] FROM docker.io/library/python:3.12@sha256:4d1caded1f729ae443eb803f26ffde7b61e696aeaef62f0  0.0s
 => => resolve docker.io/library/python:3.12@sha256:4d1caded1f729ae443eb803f26ffde7b61e696aeaef62f099abb6dd  0.0s
 => [stage-1 1/5] FROM docker.io/library/python:3.12-slim@sha256:f77ac9e44ae96ef2c90b8053ea08c31f8be030f824  0.0s
 => => resolve docker.io/library/python:3.12-slim@sha256:f77ac9e44ae96ef2c90b8053ea08c31f8be030f824196b0ae4  0.0s
 => [internal] load build context                                                                            0.0s
 => => transferring context: 12.57kB                                                                         0.0s
 => CACHED [stage-1 2/5] WORKDIR /app                                                                        0.0s
 => CACHED [stage-1 3/5] RUN useradd -m -u 1000 appuser                                                      0.0s
 => CACHED [builder 2/4] WORKDIR /app                                                                        0.0s
 => CACHED [builder 3/4] COPY requirements.txt .                                                             0.0s
 => CACHED [builder 4/4] RUN pip install --user --no-cache-dir -r requirements.txt                           0.0s
 => CACHED [stage-1 4/5] COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local       0.0s
 => [stage-1 5/5] COPY --chown=appuser:appuser . .                                                           0.0s
 => exporting to image                                                                                       0.1s
 => => exporting layers                                                                                      0.0s
 => => exporting manifest sha256:d032c1565fa4faae9e0fb517318b732f5a70cbfc540d245f13d652c42b4b0154            0.0s
 => => exporting config sha256:b0bdeff04f0796a7399b2d65fedc17d294cc7b6524a396cc16f99970ad1bc4fc              0.0s
 => => exporting attestation manifest sha256:359869091962a184ee11e575af3955e58fe614f9c60e06ee5dddd3dd472f88  0.0s
 => => exporting manifest list sha256:2bdb63781a654dab084e41809120865a2345259476978083a01e54cca8008cc9       0.0s
 => => naming to docker.io/library/catalogo-api:v1                                                           0.0s
 => => unpacking to docker.io/library/catalogo-api:v1                                                        0.0s
```

### b) Modificación del manifiesto (`backend/requirements.txt`)

Salida parcial de la reconstrucción:

```log
[+] Building 15.0s (15/15) FINISHED                                                                docker:default
 => [internal] load build definition from Dockerfile                                                         0.0s
 => => transferring dockerfile: 594B                                                                         0.0s
 => [internal] load metadata for docker.io/library/python:3.12-slim                                          1.0s
 => [internal] load metadata for docker.io/library/python:3.12                                               1.0s
 => [internal] load .dockerignore                                                                            0.0s
 => => transferring context: 74B                                                                             0.0s
 => [builder 1/4] FROM docker.io/library/python:3.12@sha256:4d1caded1f729ae443eb803f26ffde7b61e696aeaef62f0  0.0s
 => => resolve docker.io/library/python:3.12@sha256:4d1caded1f729ae443eb803f26ffde7b61e696aeaef62f099abb6dd  0.0s
 => [stage-1 1/5] FROM docker.io/library/python:3.12-slim@sha256:f77ac9e44ae96ef2c90b8053ea08c31f8be030f824  0.0s
 => => resolve docker.io/library/python:3.12-slim@sha256:f77ac9e44ae96ef2c90b8053ea08c31f8be030f824196b0ae4  0.0s
 => [internal] load build context                                                                            0.0s
 => => transferring context: 764B                                                                            0.0s
 => CACHED [builder 2/4] WORKDIR /app                                                                        0.0s
 => [builder 3/4] COPY requirements.txt .                                                                    0.0s
 => [builder 4/4] RUN pip install --user --no-cache-dir -r requirements.txt                                 12.2s
 => CACHED [stage-1 2/5] WORKDIR /app                                                                        0.0s 
 => CACHED [stage-1 3/5] RUN useradd -m -u 1000 appuser                                                      0.0s 
 => [stage-1 4/5] COPY --from=builder --chown=appuser:appuser /root/.local /home/appuser/.local              0.1s 
 => [stage-1 5/5] COPY --chown=appuser:appuser . .                                                           0.0s 
 => exporting to image                                                                                       1.4s 
 => => exporting layers                                                                                      1.2s 
 => => exporting manifest sha256:4b40b7f67bf67077d9bd09fab0851276476cac0e5e7d70616a763f169f05f898            0.0s
 => => exporting config sha256:62c10bcc45909ac7a31a28820171cc5a03dfef6d7f5c6cb45dc05a3d67f9c1fc              0.0s
 => => exporting attestation manifest sha256:d55fa46495176f31218241128051775fa5829495e607192442828a87cc7f2a  0.0s
 => => exporting manifest list sha256:eac245ed259163b3a8110c4575518b0a0eedb2ae78f90d7ff7b116e95ee0611a       0.0s
 => => naming to docker.io/library/catalogo-api:v1                                                           0.0s
 => => unpacking to docker.io/library/catalogo-api:v1                                                        0.2s
```

### Explicación técnica del orden de instrucciones

Docker procesa las capas secuencialmente e invalida la caché a partir de la primera instrucción `COPY` o `ADD` cuyo contenido cambie. Al colocar `COPY requirements.txt` e instalar las dependencias antes de copiar el código fuente de la aplicación, los cambios frecuentes en la lógica `.py` no invalidan la capa pesada de instalación de librerías.

---

## 8. Publicación en Docker Hub

Imágenes etiquetadas y publicadas en el registro oficial:

```bash
docker tag catalogo-api:v1 keze87/catalogo-api:v1
docker tag catalogo-api:v2 keze87/catalogo-api:v2
docker tag catalogo-frontend:v1 keze87/catalogo-frontend:v1

docker push keze87/catalogo-api:v1
docker push keze87/catalogo-api:v2
docker push keze87/catalogo-frontend:v1
```

### Registro de URLs y nombres finales publicados

* `docker.io/keze87/catalogo-api:v1`

* `docker.io/keze87/catalogo-api:v2`

* `docker.io/keze87/catalogo-frontend:v1`
