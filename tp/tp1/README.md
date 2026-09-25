# Informe - Trabajo Práctico N° 1

**Asignatura:** Contenedores, Docker y Orquestación con Kubernetes  
**Variante:** C - catalogo-app (React + Vite, Python + FastAPI, MongoDB 7)  
**Alumno:** César Herrera

---

## 1. Repositorio del Proyecto

* **Repositorio de la aplicación:** `https://github.com/keze87/catalogo-app`
* **Forks base de origen:**
  * Backend: `https://github.com/keze87/app-catalogo-backend`
  * Frontend: `https://github.com/keze87/app-catalogo-frontend`

Estructura de archivos verificada en la raíz del repositorio:

```text
catalogo-app/
├── .gitignore
├── README.md
├── tp/
├── backend/
└── frontend/
```

Contenido verificado de `.gitignore`:

```gitignore
node_modules/
__pycache__/
.env
```

---

## 2. Arquitectura de Tres Capas

La documentación del sistema fue incorporada en el archivo `README.md` principal:

```markdown
# Arquitectura

    navegador ──► catalogo-frontend ──────► catalogo-api ──────► catalogo-db
                  React + Vite              Python + FastAPI     MongoDB 7
                  nginx :8080               :8000                :27017
                  (host 3000)               (host 8000, solo     (sin puerto
                                             depuración)          publicado)

| Capa | Imagen | Puerto interno | Puerto publicado |
| --- | --- | --- | --- |
| `catalogo-frontend` | React + Vite servido por nginx | 8080 | 3000 |
| `catalogo-api` | Python + FastAPI | 8000 | 8000 (solo depuración) |
| `catalogo-db` | MongoDB 7 | 27017 | — |

El frontend es el único punto de entrada: nadie le habla a la base directamente, y a la API le habla el frontend.
```

---

## 3. Ejecución del Motor sin Configuración

Comando ejecutado:

```bash
docker run -d --name catalogo-db mongo:7
```

### a) Estado del contenedor (`docker ps`)

```text
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS       NAMES
e863483d96e4   mongo:7   "docker-entrypoint.s…"   11 seconds ago   Up 10 seconds   27017/tcp   catalogo-db
```

### b) Listado de bases con `mongosh` sin credenciales

```bash
docker exec -it catalogo-db mongosh --eval "show dbs"
```

**Salida obtenida:**

```text
admin    8.00 KiB
config  12.00 KiB
local    8.00 KiB
```

### c) Análisis de riesgo

Esta condición es significativamente más riesgosa que una falla explícita porque produce silenciosa de seguridad. Si el contenedor fallara al iniciar sin contraseñas obligaría al administrador a corregir la configuración antes de desplegar, al permanecer activo sin advertencias evidentes, deja el motor expuesto a accesos no autorizados y filtración de datos.

---

## 4. Configuración con Variables de Entorno Oficiales

Variables de entorno identificadas en Docker Hub para crear el usuario raíz:

* `MONGO_INITDB_ROOT_USERNAME`
* `MONGO_INITDB_ROOT_PASSWORD`

Comando ejecutado:

```bash
docker rm -f catalogo-db
docker run -d --name catalogo-db \
  -e MONGO_INITDB_ROOT_USERNAME=catalogo_user \
  -e MONGO_INITDB_ROOT_PASSWORD=catalogo_pass \
  mongo:7
```

### Registro de inicio (`docker logs -f catalogo-db`)

```json
{"t":{"$date":"2026-09-25T00:06:25.554+00:00"},"s":"I",  "c":"NETWORK",  "id":23016,   "ctx":"listener","msg":"Waiting for connections","attr":{"port":27017,"ssl":"off"}}
```

---

## 5. Verificación de Autenticación Habilitada

### a) Operación administrativa `ping` sin credenciales

```bash
docker exec -it catalogo-db mongosh --eval "db.adminCommand({ ping: 1 })"
```

**Salida obtenida:**

```json
{ ok: 1 }
```

### b) Listado de bases sin credenciales

```bash
docker exec -it catalogo-db mongosh --eval "show dbs"
```

Salida y mensaje de error obtenido:

```text
MongoServerError: Command listDatabases requires authentication
```

### Justificación de la diferencia

El comando `ping` está diseñado para comprobaciones básicas de vida y conectividad a nivel de socket de red sin exponer datos ni modificar el estado del sistema, por lo que MongoDB lo permite sin una sesión autenticada. Pero la operación `listDatabases` consulta el catálogo y los metadatos internos de las bases de datos de la instancia, una acción protegida que requiere autorización de lectura administrativa obligatoria.

---

## 6. Autenticación y `authSource`

### a) Conexión sin especificar base de autenticación

```bash
docker exec -it catalogo-db mongosh \
  "mongodb://catalogo_user:catalogo_pass@localhost:27017/catalogo" \
  --eval "show dbs"
```

Salida obtenida (Fallo de autenticación):

```text
MongoServerError: Authentication failed.
```

### b) Conexión con parámetro `?authSource=admin`

```bash
docker exec -it catalogo-db mongosh \
  "mongodb://catalogo_user:catalogo_pass@localhost:27017/catalogo?authSource=admin" \
  --eval "show dbs"
```

Salida obtenida (Acceso concedido):

```text
admin   100.00 KiB
config   12.00 KiB
local    72.00 KiB
```

### Explicación técnica

El usuario definido a través de las variables de entorno oficiales se almacena físicamente dentro de la base de datos de sistema `admin`. Al no declarar explícitamente `?authSource=admin` en la URI de conexión, el cliente asume por defecto que las credenciales deben verificarse en la base declarada en la ruta (`catalogo`), donde el usuario no existe, produciendo un rechazo inmediato de autenticación.

---

## 7. Inspección de Variables de Entorno

Comando ejecutado:

```bash
docker inspect --format '{{json .Config.Env}}' catalogo-db
```

Salida obtenida:

```json
["MONGO_INITDB_ROOT_USERNAME=catalogo_user","MONGO_INITDB_ROOT_PASSWORD=catalogo_pass","PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","GOSU_VERSION=1.19","JSYAML_VERSION=3.13.1","JSYAML_CHECKSUM=662e32319bdd378e91f67578e56a34954b0a2e33aca11d70ab9f4826af24b941","MONGO_PACKAGE=mongodb-org","MONGO_REPO=repo.mongodb.org","MONGO_MAJOR=7.0","MONGO_VERSION=7.0.43","HOME=/data/db"]
```

### Implicancia de seguridad

Pasar credenciales como variables de entorno directamente en el comando de ejecución deja los secretos en texto plano legibles dentro de los metadatos del demonio Docker, exponiéndolos ante cualquier usuario o proceso con acceso al socket local o inspección del host.

---

## 8. Ciclo de Vida y Persistencia sin Volúmenes

### a) Inserción de documento de prueba en la base `catalogo`

```bash
docker exec -it catalogo-db mongosh \
  "mongodb://catalogo_user:catalogo_pass@localhost:27017/catalogo?authSource=admin" \
  --eval 'db.productos.insertOne({ nombre: "Teclado", precio: 1234 })'
```

**Salida:**

```json
{
  acknowledged: true,
  insertedId: ObjectId('6ab5c2ae3cd61c4e6878bf2a')
}
```

### b) Verificación inicial de documentos existentes

```bash
docker exec -it catalogo-db mongosh \
  "mongodb://catalogo_user:catalogo_pass@localhost:27017/catalogo?authSource=admin" \
  --eval 'db.productos.countDocuments()'
```

**Salida:**

```text
1
```

### c) Eliminación, recreación y nuevo conteo

```bash
docker rm -f catalogo-db
docker run -d --name catalogo-db \
  -e MONGO_INITDB_ROOT_USERNAME=catalogo_user \
  -e MONGO_INITDB_ROOT_PASSWORD=catalogo_pass \
  mongo:7
docker exec -it catalogo-db mongosh \
  "mongodb://catalogo_user:catalogo_pass@localhost:27017/catalogo?authSource=admin" \
  --eval 'db.productos.countDocuments()'
```

Salida del conteo tras recrear el contenedor:

```text
0
```

### Explicación del resultado

Al no haber configurado un volumen de almacenamiento gestionado, los archivos de la base de datos se escribieron únicamente en la capa modificable temporal del contenedor. Cuando el contenedor fue destruido mediante `docker rm -f`, dicha capa de escritura fue eliminada en su totalidad junto con los datos.
