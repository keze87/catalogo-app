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

## Consideraciones de Base de Datos y Conexión

### Autenticación y `authSource`
El usuario administrador inicial (`catalogo_user`), configurado mediante las variables de entorno `MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD` de la imagen oficial `mongo:7`, reside físicamente en la base de datos de autenticación del sistema: `admin`.

Al construir la cadena de conexión hacia la base propia de la aplicación (`catalogo`), es **obligatorio** especificar el parámetro de consulta `?authSource=admin`:

```text
mongodb://catalogo_user:catalogo_pass@<host>:27017/catalogo?authSource=admin
```
