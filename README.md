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
