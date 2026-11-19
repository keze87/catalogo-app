# Arquitectura — `catalogo-app` (Variante C)

## 1. Preguntas obligatorias del curso

### a) Configuración del frontend en tiempo de ejecución (`API_HOST`) frente a tiempo de construcción (`VITE_API_URL`)

En una aplicación React empaquetada con Vite, las variables con prefijo `VITE_*` (`import.meta.env.VITE_API_URL`) se resuelven estáticamente durante la fase de compilación (`npm run build` en la etapa `builder` de `frontend/Dockerfile`). El empaquetador incrusta esos valores como literales dentro del bundle JavaScript resultante en `dist/`. En consecuencia, una vez construida la imagen de contenedor, el parámetro queda fijado en el artefacto y no responde a variables de entorno inyectadas al iniciar el contenedor.

Si se hubiera adoptado la configuración en tiempo de construcción, promover la aplicación desde desarrollo local (Docker Compose) hacia un entorno de pruebas o producción (Kubernetes) exigiría modificar la URL objetivo, recompilar el frontend, reconstruir la imagen Docker, reetiquetarla y publicarla nuevamente en el registro por cada entorno. Esto invalida la portabilidad de un único artefacto inmutable entre entornos.

Para evitarlo, el cliente HTTP (`frontend/src/api.js`) realiza todas las solicitudes sobre la ruta relativa `/api`. Al iniciar el contenedor basado en `nginxinc/nginx-unprivileged:1.27-alpine`, el entrypoint ejecuta `envsubst` sobre `/etc/nginx/templates/default.conf.template` restringido mediante `NGINX_ENVSUBST_FILTER=^API_`, de modo que sustituya únicamente `${API_HOST}` y `${API_PORT}` sin alterar variables internas de Nginx como `$uri`, `$host` o `$proxy_add_x_forwarded_for`. De esta manera, Nginx resuelve `proxy_pass http://${API_HOST}:${API_PORT}/;` en tiempo de arranque y una única imagen (`keze87/catalogo-frontend:v1`) opera sin modificaciones tanto en Compose como en Kubernetes.

### b) Ausencia de CORS y requisitos ante comunicación directa con la API

CORS (*Cross-Origin Resource Sharing*) es un control aplicado por el navegador web bajo la política de mismo origen (*Same-Origin Policy*), que impide que el código JavaScript ejecutado desde un origen `(esquema, host, puerto)` lea respuestas de otro origen salvo autorización explícita mediante cabeceras HTTP.

En esta arquitectura, desde la perspectiva del navegador todo el tráfico pertenece al mismo origen (`http://localhost:3000` en Compose o `http://catalogo.local` detrás del `Ingress` en Kubernetes). Tanto los archivos estáticos de la SPA como las llamadas a `/api/*` se dirigen al mismo host y puerto. El reenvío hacia `http://catalogo-api:8000/` ocurre entre servidores a través de Nginx, fuera del ámbito de la política de mismo origen del navegador.

Si la aplicación de página única se comunicara directamente con la API (por ejemplo, sirviendo el frontend en `http://catalogo.local` y consultando `http://api.catalogo.local`), sería necesario:
- Incorporar `CORSMiddleware` en `backend/main.py` declarando los orígenes permitidos (`allow_origins`), los métodos HTTP utilizados (`GET`, `POST`, `PUT`, `DELETE`, `OPTIONS`) y las cabeceras (`Content-Type`), dado que el envío de JSON en `frontend/src/api.js` dispara solicitudes *preflight* `OPTIONS` antes de cada operación de escritura.
- Exponer la API públicamente fuera del clúster en lugar de mantenerla detrás de un `Service` interno de tipo `ClusterIP`, e inyectar la URL externa correspondiente al entorno en el cliente web.

### c) Autoescalado exclusivo en `catalogo-api`

Entre los doce objetos del despliegue, únicamente `catalogo-api` cuenta con un `HorizontalPodAutoscaler` (`catalogo-api-hpa`, configurado entre 2 y 6 réplicas con un umbral objetivo de 50 % de utilización sobre `requests.cpu: 100m`), ya que es el único componente que combina ejecución sin estado (*stateless*) con carga significativa de cómputo: validación de esquemas con Pydantic, decodificación Base64, consultas a MongoDB y lectura/entrega de binarios en `/productos/{id}/imagen`.

Asignar un HPA a `catalogo-db` provocaría fallas operativas graves porque la base está desplegada como un `Deployment` respaldado por un único `PersistentVolumeClaim` (`catalogo-db-pvc`) en modo `ReadWriteOnce` (`RWO`) y sin *replica set*:
- Si el planificador ubica la nueva réplica en otro nodo, el Pod quedará bloqueado en `ContainerCreating` al no poder montarse un volumen `RWO` en múltiples nodos simultáneamente.
- Si se programa en el mismo nodo, dos procesos `mongod` intentarían adquirir el bloqueo exclusivo (`mongod.lock`) sobre el mismo directorio `/data/db` de WiredTiger, provocando el fallo del segundo proceso o riesgo de corrupción de datos.
- Aun utilizando volúmenes separados por réplica, al carecer de replicación coordinada, cada instancia mantendría un estado divergente e inconsistente.

Por su parte, `catalogo-frontend` tampoco requiere autoescalado: Nginx entrega archivos estáticos y actúa como proxy inverso asíncrono con un consumo marginal de CPU (alrededor de `1m` por réplica incluso durante la prueba de carga). Mantener `replicas: 2` resulta suficiente para garantizar disponibilidad ante fallos o reinicios de un Pod.

---

## 2. Pregunta propia de la Variante C: transición de una a tres réplicas de la API

Con una única réplica de `catalogo-api`, todas las peticiones son atendidas por el mismo proceso, compartiendo el mismo espacio de memoria y sistema de archivos local. Al escalar a **tres réplicas** balanceadas por el `Service` `catalogo-api`, **deja de estar garantizada la afinidad de estado local**: dos solicitudes consecutivas pueden recaer en Pods diferentes.

La implicancia directa es que **un archivo escrito en el sistema de archivos de un Pod no es visible para ninguna otra réplica y se pierde de forma definitiva cuando ese Pod es destruido o recreado** (ante un fallo, una actualización progresiva o una reducción de réplicas por el HPA).

Por este motivo, las imágenes del catálogo no se almacenan en el sistema de archivos del contenedor de `catalogo-api`, sino como campo binario (`imagen_datos`) dentro del documento correspondiente en MongoDB, acompañado de `imagen_tipo` e `imagen_hash` (`backend/main.py`). Esto permite que la API permanezca completamente sin estado (*stateless*) y que cualquiera de las réplicas sirva `GET /productos/{id}/imagen` de manera indistinta.

Esta condición se verificó experimentalmente en la prueba correspondiente al TP6 (documentada en la sección 5 de [`INFORME.md`](INFORME.md)): con múltiples réplicas activas, el `Service` balancea solicitudes devolviendo `200`, mientras que un archivo `/tmp/foto.bin` creado en una réplica mediante `kubectl exec` resulta inexistente al inspeccionar los demás Pods (`No such file or directory`).

Como corolario, si la aplicación hubiera incorporado un carrito de compras mantenido en la sesión en memoria del servidor, tampoco habría sobrevivido a la segunda réplica: una operación registrada en la réplica A no existiría en la memoria de la réplica B, produciendo inconsistencias para el usuario salvo que el estado de sesión se externalizara en un almacén compartido o en el cliente.

---

## 3. Dificultades propias de MongoDB observadas durante el desarrollo

### a) Parámetro `authSource=admin` en la URI de conexión
- **Manifestación:** Al iniciar `catalogo-api` apuntando a la base `catalogodb` con las credenciales definidas mediante `MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD`, la conexión fallaba con `Authentication failed` pese a que usuario y contraseña eran correctos.
- **Diagnóstico:** La imagen oficial `mongo:7` crea el usuario root en la base de autenticación del sistema `admin`, no en la base de la aplicación (`catalogodb`). Al omitir `authSource`, el controlador PyMongo intenta validar las credenciales contra `catalogodb`, rechazando el acceso.
- **Resolución:** En `backend/main.py` se construye la cadena de conexión anexando explícitamente `?authSource=admin`. En Kubernetes se optó por inyectar los componentes de conexión por separado (`DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`) para reutilizar un único `Secret` (`catalogo-db-secret`) en ambos servicios sin duplicar credenciales.

### b) Falso positivo de disponibilidad en el healthcheck `ping`
- **Manifestación:** En Docker Compose, `catalogo-db` alcanzaba el estado `healthy` mediante `mongosh --quiet --eval "db.adminCommand('ping').ok"`, pero `catalogo-api` podía fallar en la autenticación si el volumen persistente `catalogo-db-data` conservaba credenciales de una inicialización previa (o si en despliegues manuales se omitían las variables `MONGO_INITDB_ROOT_*`, caso en el cual `mongo:7` inicia sin autenticación).
- **Diagnóstico:** El comando `ping` de MongoDB no requiere autenticación. Esto evita exponer la contraseña en la definición del sondeo (visible mediante `docker inspect`), pero implica que `healthy` únicamente verifica conectividad TCP/protocolo hacia `mongod`, no la validez de las credenciales configuradas.
- **Resolución:** Se mantuvo el `ping` sin credenciales en la capa de datos documentando su alcance, se implementó `/health` en la API ejecutando un `ping` autenticado desde PyMongo (devolviendo `503` si falla la autenticación o la red), y se utilizó `docker compose down -v` cuando fue necesario reinicializar el volumen desde cero.

### c) Siembra idempotente concurrente y prefijo `ROOT_PATH=/api` detrás del proxy
- **Manifestación:** Con múltiples réplicas de `catalogo-api` iniciando en paralelo sobre una base persistente, una inserción simple del catálogo de demostración generaría duplicados o fallas por clave única. Adicionalmente, al acceder a `http://catalogo.local/api/docs` detrás de Nginx, Swagger UI fallaba al solicitar `/openapi.json` en la raíz en lugar de `/api/openapi.json`.
- **Diagnóstico:** Cada réplica ejecuta `lifespan` al arrancar, y la directiva `proxy_pass http://${API_HOST}:${API_PORT}/;` recorta el prefijo `/api`, ocultándole a FastAPI la ruta pública bajo la cual es consumido.
- **Resolución:** Se definió un índice único sobre `sku` y se implementó `sembrar_catalogo()` mediante `update_one(..., upsert=True)`, garantizando idempotencia ante ejecuciones concurrentes. Para la documentación interactiva se configuró `ROOT_PATH: "/api"` en `catalogo-api-config`.

---

## 4. Aspectos pendientes para un entorno de producción real

El despliegue actual cubre los objetivos académicos del proyecto, pero un entorno productivo requeriría incorporar las siguientes capacidades:

- **Observabilidad:** `metrics-server` solo provee lecturas instantáneas de CPU y memoria para el HPA. En producción se requiere recolección histórica de métricas y alertas (Prometheus y Grafana), agregación centralizada de logs (Loki o Elasticsearch), trazabilidad distribuida (OpenTelemetry) y definición explícita de `readinessProbe` y `livenessProbe` en los `Deployments` aprovechando `/health` y `/healthz`.
- **Integración y despliegue continuos (CI/CD):** pipelines automatizados que ejecuten pruebas, validaciones estáticas (`helm lint`), escaneo bloqueante de vulnerabilidades sobre las imágenes base (en la auditoría del TP4 se registraron más de 130 vulnerabilidades en las imágenes), publicación con etiquetas inmutables y despliegue declarativo.
- **Gestión externa de secretos y endurecimiento de seguridad:** en este repositorio `k8s/secret-catalogo-db.yaml` y `values.yaml` contienen credenciales de laboratorio (`catalogo_user` / `catalogo_pass`). Dado que los `Secret` nativos de Kubernetes solo utilizan codificación Base64, en producción debe habilitarse *Encryption at Rest* en `etcd` e integrarse un gestor externo (HashiCorp Vault, Sealed Secrets o External Secrets Operator). Asimismo, corresponde utilizar un usuario con privilegios acotados a `catalogodb` (en lugar del usuario root), habilitar TLS en el Ingress (`cert-manager`) y aplicar `NetworkPolicies` restringiendo el puerto `27017` únicamente a `catalogo-api`.
- **Copias de respaldo y almacenamiento de objetos:** automatizar respaldos periódicos (`mongodump` mediante `CronJob` o snapshots de volumen) hacia almacenamiento externo inmutable con pruebas periódicas de restauración. Para catálogos de mayor volumen, las imágenes deberían delegarse a un Object Storage (S3/MinIO) para no sobrecargar los documentos BSON ni el *working set* de WiredTiger.
- **Replica Set de MongoDB frente a `Deployment` con un volumen:** un `Deployment` con `replicas: 1` y un único PVC constituye un punto único de fallo y puede presentar bloqueos de montaje `RWO` durante actualizaciones. En un entorno productivo se debe desplegar un **Replica Set de al menos tres miembros** mediante un `StatefulSet` (o un operador oficial de MongoDB / servicio gestionado), donde cada instancia disponga de identidad de red estable (`mongo-0`, `mongo-1`, `mongo-2`) y su propio PVC vía `volumeClaimTemplates`, asegurando alta disponibilidad y *failover* automático.
