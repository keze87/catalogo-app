# Trabajo Práctico N° 7: Configuración Externalizada, Persistencia e Ingress

**Asignatura:** Contenedores, Docker y Orquestación con Kubernetes
**Variante:** C - `catalogo-app` (React + Vite, Python + FastAPI, MongoDB 7)
**Alumno:** César Ezequiel Herrera

---

## 1. Externalización de Configuración: ConfigMaps (`catalogo-api-config` y `catalogo-frontend-config`)

### a) `k8s/cm-catalogo-api.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalogo-api-config
  namespace: catalogo-app
data:
  DB_HOST: "catalogo-db"
  DB_PORT: "27017"
  DB_NAME: "catalogodb"
  ROOT_PATH: "/api"
  SEED_DEMO: "true"
```

* **Nota sobre `APP_VERSION`:** No se incluye en el ConfigMap porque la versión de la aplicación viene fijada durante la construcción de la imagen (`ARG`/`ENV` en el `Dockerfile`); declararla aquí sobrescribiría el metadato propio de cada release.

### b) `k8s/cm-catalogo-frontend.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: catalogo-frontend-config
  namespace: catalogo-app
data:
  API_HOST: "catalogo-api"
  API_PORT: "8000"
```

* **Regla de diseño:** Aunque contiene únicamente dos parámetros, ningún valor de configuración literal permanece escrito dentro de los manifiestos de `Deployment`.

---

## 2. Gestión de Credenciales: Secret (`catalogo-db-secret`)

### a) Manifiesto `k8s/secret-catalogo-db.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: catalogo-db-secret
  namespace: catalogo-app
type: Opaque
stringData:
  MONGO_INITDB_ROOT_USERNAME: "catalogo_user"
  MONGO_INITDB_ROOT_PASSWORD: "catalogo_pass"
```

### b) Comprobación de ausencia de cifrado

Comando ejecutado para decodificar el valor almacenado:

```bash
kubectl get secret catalogo-db-secret -n catalogo-app \
  -o jsonpath='{.data.MONGO_INITDB_ROOT_PASSWORD}' | base64 -d; echo
```

**Salida obtenida:**

```text
catalogo_pass
```

* **Análisis de seguridad:** Un objeto `Secret` estándar en Kubernetes almacena sus datos codificados en **Base64**, lo cual es una representación reversible y **no constituye un mecanismo de cifrado**. Para que las credenciales estén realmente protegidas se requiere habilitar el **cifrado en reposo (*Encryption at Rest*) en `etcd`** o integrar un **gestor de secretos externo** (como HashiCorp Vault, Sealed Secrets o External Secrets Operator).

---

## 3. Refactorización de Deployments y Verificación de Variables

Se actualizaron los tres `Deployments` para consumir los objetos externos mediante `envFrom` y `valueFrom.secretKeyRef`, renombrando las claves del Secret (`MONGO_INITDB_ROOT_USERNAME` y `MONGO_INITDB_ROOT_PASSWORD`) hacia los nombres que espera la API (`DB_USER` y `DB_PASSWORD`), sin dejar ningún valor literal en los manifiestos.

### a) `k8s/deploy-catalogo-db.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogo-db
  namespace: catalogo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: catalogo-db
  template:
    metadata:
      labels:
        app: catalogo-db
    spec:
      containers:
        - name: mongo
          image: mongo:7
          ports:
            - containerPort: 27017
          envFrom:
            - secretRef:
                name: catalogo-db-secret
          volumeMounts:
            - name: mongo-data
              mountPath: /data/db
      volumes:
        - name: mongo-data
          persistentVolumeClaim:
            claimName: catalogo-db-pvc
```

### b) `k8s/deploy-catalogo-api.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogo-api
  namespace: catalogo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalogo-api
  template:
    metadata:
      labels:
        app: catalogo-api
    spec:
      containers:
        - name: catalogo-api
          image: keze87/catalogo-api:v1
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: catalogo-api-config
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: catalogo-db-secret
                  key: MONGO_INITDB_ROOT_USERNAME
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: catalogo-db-secret
                  key: MONGO_INITDB_ROOT_PASSWORD
```

### c) `k8s/deploy-catalogo-frontend.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalogo-frontend
  namespace: catalogo-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: catalogo-frontend
  template:
    metadata:
      labels:
        app: catalogo-frontend
    spec:
      containers:
        - name: frontend
          image: keze87/catalogo-frontend:v1
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: catalogo-frontend-config
```

### d) Verificación efectiva dentro de los contenedores

*(Nota: si a la API no le llegan `DB_USER` o `DB_PASSWORD`, utiliza valores por omisión que coinciden con Docker Compose y la aplicación sigue respondiendo igual; por eso inspeccionar `env` dentro de cada contenedor es la única verificación válida).*

```bash
# Variables en el contenedor de la API
kubectl exec deploy/catalogo-api -n catalogo-app -- \
  env | grep -E '^(DB_(HOST|PORT|NAME|USER)|ROOT_PATH)='
```

**Salida en `catalogo-api`:**

```sh
ROOT_PATH=/api
DB_HOST=catalogo-db
DB_NAME=catalogodb
DB_PORT=27017
DB_USER=catalogo_user
```

```bash
# Variables en el contenedor del Frontend
kubectl exec deploy/catalogo-frontend -n catalogo-app -- \
  env | grep '^API_'
```

**Salida en `catalogo-frontend`:**

```sh
API_HOST=catalogo-api
API_PORT=8000
```

---

## 4. Decisión Arquitectónica: Cadena de Conexión a MongoDB

* **Alternativa elegida:** Entrega de **piezas por separado** (`DB_HOST`, `DB_PORT`, `DB_NAME` desde `catalogo-api-config`, y `DB_USER`, `DB_PASSWORD` desde `catalogo-db-secret`).

* **Justificación:**

  1. **Reutilización sin duplicación:** Permite que un único `Secret` (`catalogo-db-secret`) alimente tanto al contenedor oficial de MongoDB (que exige `MONGO_INITDB_ROOT_*`) como a FastAPI (mediante mapeo de claves), evitando mantener dos secretos distintos con las mismas credenciales.
  2. **Gestión automática de autenticación:** Al recibir los componentes desacoplados, el backend construye dinámicamente la URI y anexa por sí mismo el parámetro `?authSource=admin`, eliminando el riesgo de errores `Authentication failed` por omisión manual en una cadena completa.

---

## 5. Persistencia de Datos (`catalogo-db-pvc`)

### a) Manifiesto `k8s/pvc-catalogo-db.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: catalogo-db-pvc
  namespace: catalogo-app
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

### b) Estados del PVC y PV

```bash
kubectl get pvc,pv -n catalogo-app
```

* **Estado inmediatamente tras aplicar el PVC (antes de montar en el Pod):**

```text
NAME                                    STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/catalogo-db-pvc   Pending                                      local-path     <unset>          0s
```

* **Estado tras actualizar `deploy-catalogo-db` y crear el Pod:**

```text
NAME                                    STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/catalogo-db-pvc   Bound    pvc-f91b6758-b98e-44df-808a-5d7b4db83992   1Gi        RWO    local-path     <unset>                 90s
```

### c) Diferencia con MySQL (`subPath`) y verificación de permisos

* **Ausencia de `subPath`:** A diferencia de MySQL (que aborta su inicialización si encuentra subdirectorios como `lost+found` en el punto de montaje), **MongoDB tolera que `/data/db` no esté completamente vacío**, por lo que el volumen se monta directamente sin requerir `subPath`.
* **Inspección de permisos del directorio:**

```bash
kubectl exec deploy/catalogo-db -n catalogo-app -- ls -ld /data/db
```

**Salida obtenida:**

```text
drwxrwxrwx 5 mongodb root 4096 Nov 13 17:42 /data/db
```

---

## 6. Comprobación de Persistencia Efectiva

1. Se accedió a la aplicación en `http://172.16.21.17:30302` y se cargó manualmente un producto con imagen.
2. Se eliminó el Pod de la base de datos y se aguardó su recreación automática por el `Deployment`:

```bash
kubectl delete pod -l app=catalogo-db -n catalogo-app
kubectl get pods -n catalogo-app -w
```

3. Se consultó el catálogo tras la recuperación del Pod:

```bash
curl -s http://172.16.21.17:30302/api/productos | head -c 300; echo
```

**Salida obtenida:**

```json
[{"sku":1,"nombre":"11","precio":1.0,"categoria":"General","stock":1,"descripcion":"11","imagen_tipo":"image/jpeg","imagen_hash":"c9e1c99c01d2d401","creado_en":"2026-11-13T18:06:10.764000","id":"6af75192d5e423664919b5df","imagen":true}]
```

* **Falso positivo del catálogo de demostración:** Los productos de ejemplo (`SEED_DEMO=true`) se siembran de manera idempotente cada vez que arranca la API, por lo que reaparecen siempre aunque los datos se hayan perdido. La supervivencia del **producto cargado manualmente con su imagen** es la prueba válida de que los datos persisten en el `PersistentVolume`, cerrando definitivamente la limitación declarada en el TP6.

---

## 7. Exposición mediante Ingress (`catalogo-app-ingress`)

### a) Manifiesto `k8s/ingress-catalogo-app.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catalogo-app-ingress
  namespace: catalogo-app
spec:
  rules:
    - host: catalogo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: catalogo-frontend
                port:
                  number: 8080
```

* **IP Externa del Ingress Controller (`nginx-ingress`):** `172.16.21.212`
* **Registro en `/etc/hosts`:** `172.16.21.212 catalogo.local`

**Estado del Ingress:**

```bash
kubectl get ingress -n catalogo-app
```

```text
NAME                   CLASS   HOSTS            ADDRESS         PORTS   AGE
catalogo-app-ingress   nginx   catalogo.local   172.16.21.212   80      32s
```

### b) Justificación de regla única (`/`)

No se definió una regla `/api` en el `Ingress` porque el servidor **Nginx embebido dentro del contenedor `catalogo-frontend` ya realiza el enrutamiento inverso hacia `catalogo-api:8000`**. Duplicar esa regla en el Ingress crearía dos fuentes de verdad para el mismo enrutamiento y rompería la paridad arquitectónica con Docker Compose.

### c) Verificación válida (`/api/productos`) vs Falso positivo SPA (`/productos`)

* **Consulta a `http://catalogo.local/api/productos`:**

```bash
curl -s http://catalogo.local/api/productos | head -c 200; echo
```

**Resultado:** Devuelve `200 OK` con el arreglo **JSON** real proveniente de FastAPI.

* **Consulta a `http://catalogo.local/productos`:**

```bash
curl -i -s http://catalogo.local/productos | head -n 12
```

**Resultado:** Devuelve `200 OK` con el documento **`index.html`** de React. Esto ocurre por la directiva `try_files $uri $uri/ /index.html;` configurada en Nginx para soportar rutas del lado del cliente en aplicaciones de página única (SPA), constituyendo un falso positivo silencioso si se usa como prueba de conectividad con el backend.

---

## 8. Documentación Interactiva detrás del Ingress (`/api/docs`)

* **URL verificada en navegador:** `http://catalogo.local/api/docs`
* **Resultado observado:** La interfaz de Swagger UI carga completamente y ejecuta peticiones contra los endpoints de la API.
* **Explicación:** Confirma que la variable `ROOT_PATH=/api` llegó efectivamente desde `catalogo-api-config` hasta FastAPI a través de la nueva capa de indirección, permitiendo que el esquema OpenAPI genere sus rutas con el prefijo `/api` esperado por el reverse proxy.

---

## 9. Inventario Actual de Objetos en el Clúster (11 de 12)

| # | Nombre del Objeto | Tipo (`Kind`) | Función en la Arquitectura |
| --- | --- | --- | --- |
| 1 | `catalogo-db` | `Deployment` | Gestiona la réplica única del motor MongoDB 7 con almacenamiento persistente. |
| 2 | `catalogo-api` | `Deployment` | Mantiene las réplicas sin estado del backend FastAPI. |
| 3 | `catalogo-frontend` | `Deployment` | Administra las réplicas del servidor Nginx con el build estático de React y proxy `/api`. |
| 4 | `catalogo-db` | `Service` (`ClusterIP`) | Provee resolución DNS interna estable en el puerto `27017` hacia MongoDB. |
| 5 | `catalogo-api` | `Service` (`ClusterIP`) | Balancea internamente las peticiones HTTP en el puerto `8000` hacia las réplicas de la API. |
| 6 | `catalogo-frontend` | `Service` (`NodePort`) | Expone el frontend en el puerto fijo `30302` de los nodos y recibe tráfico del Ingress. |
| 7 | `catalogo-api-config` | `ConfigMap` | Externaliza los parámetros no sensibles del backend (`DB_HOST`, `DB_PORT`, `DB_NAME`, `ROOT_PATH`, `SEED_DEMO`). |
| 8 | `catalogo-frontend-config` | `ConfigMap` | Externaliza el host y puerto de destino de la API (`API_HOST`, `API_PORT`) para Nginx. |
| 9 | `catalogo-db-secret` | `Secret` | Almacena las credenciales de administrador de MongoDB consumidas por la base y la API. |
| 10 | `catalogo-db-pvc` | `PersistentVolumeClaim` | Solicita y vincula `1Gi` de almacenamiento persistente en modo `ReadWriteOnce` para `/data/db`. |
| 11 | `catalogo-app-ingress` | `Ingress` | Enruta el tráfico HTTP entrante del host `catalogo.local` (puerto 80) hacia `catalogo-frontend`. |

<!-- *(Nota: El objeto previsto en el proyecto integrador corresponde al autoescalador horizontal de Pods —`HorizontalPodAutoscaler`—).* -->
