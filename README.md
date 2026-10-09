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

> **Limitación:** El orden actual reside exclusivamente en la coordinación manual del operador, careciendo de automatización declarativa para la gestión de dependencias y tiempos de espera.
