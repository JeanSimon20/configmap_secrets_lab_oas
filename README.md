# Guía de Laboratorio Práctico: S08 | AP6 | Exposición de Servicios y Configuración Segura

**Rol:** DevOps Engineer Senior & Diseñador de Contenidos Académicos  
**Entorno:** Minikube (Kubernetes local) + `kubectl`  
**Stack Tecnológico:** Node.js (`node:20-alpine`) + MongoDB (`mongo:7.0`)  

---

## 🎯 Objetivo del Laboratorio

Desplegar una arquitectura multicapa compuesta por un **Backend en Node.js** conectado a una base de datos **MongoDB** con almacenamiento persistente en Kubernetes (**Minikube**), aplicando el principio de **12-Factor App (Factor III: Config)** para separar completamente el código de la configuración mediante `ConfigMaps` y `Secrets`, sin hardcodear credenciales ni cadenas de conexión, y exponiendo únicamente la capa de aplicación mediante un servicio `NodePort`.

---

## 🏗️ Arquitectura de la Solución

```
[ Cliente: Navegador / curl ]
              │
              ▼ (HTTP: minikube IP : NodePort 30080)
┌──────────────────────────────────────────────────────────┐
│ Clúster Kubernetes (Minikube)                            │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Service: backend-service (Tipo: NodePort -> :3000) │  │
│  └─────────────────────────┬──────────────────────────┘  │
│                            ▼                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Deployment: backend-deployment (2 Réplicas Node.js)│  │
│  │  ├── envFrom  ◄── ConfigMap: app-config            │  │
│  │  │                (DB_HOST, DB_NAME, PORT)         │  │
│  │  └── valueFrom ◄── Secret: db-secret               │  │
│  │                   (DB_USER, DB_PASSWORD)           │  │
│  └─────────────────────────┬──────────────────────────┘  │
│                            │ (DNS interno: mongo-service:27017)
│                            ▼                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Service: mongo-service (Tipo: ClusterIP -> :27017) │  │
│  └─────────────────────────┬──────────────────────────┘  │
│                            ▼                             │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Deployment: mongodb-deployment (1 Réplica Mongo 7) │  │
│  │  └── /data/db ◄── PVC: mongo-pvc (1Gi)             │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## 0. Requisitos Previos: Inicio y Verificación de Minikube

Antes de crear los recursos, asegúrate de tener **Docker Desktop** (o el driver configurado) en ejecución e inicia tu clúster local de Minikube.

### Comandos de terminal

```bash
# 1. Iniciar el clúster local de Minikube
minikube start --cpus=2 --memory=4096

# 2. Verificar el estado de los componentes de Minikube
minikube status

# 3. Comprobar que kubectl apunta al contexto de Minikube y el nodo está Ready
kubectl config current-context
kubectl get nodes
```

**Salida esperada (`minikube status`):**
```text
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

---

## Paso 1: Creación de ConfigMaps & Secrets

En Kubernetes desacoplamos la configuración en dos objetos nativos:
- **`ConfigMap`**: Para datos de configuración **no sensibles** (`DB_HOST`, `DB_NAME`, `PORT`).
- **`Secret`**: Para datos **sensibles** como credenciales (`DB_USER`, `DB_PASSWORD`), almacenados en formato `Base64`.

### 1.1 Creación del ConfigMap (`app-config`)

#### Opción A: Imperativa desde CLI (Pruebas rápidas)
```bash
kubectl create configmap app-config \
  --from-literal=DB_HOST=mongo-service \
  --from-literal=DB_NAME=academiadb \
  --from-literal=PORT=3000
```

*(Si creaste el ConfigMap por CLI y deseas continuar usando el enfoque declarivo YAML recomendado en DevOps, elimínalo antes de aplicar el archivo: `kubectl delete configmap app-config`).*

#### Opción B: Declarativa desde Archivo YAML (`01-configmap.yaml`)

Archivo `01-configmap.yaml`:
```yaml
# ==============================================================================
# ARCHIVO: 01-configmap.yaml
# DESCRIPCIÓN: Define parámetros de configuración NO sensibles para la aplicación
# ==============================================================================
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  labels:
    app: backend
    tier: configuration
data:
  # Nombre del Service ClusterIP de MongoDB (resolución vía CoreDNS interno)
  DB_HOST: "mongo-service"
  # Nombre de la base de datos de la aplicación
  DB_NAME: "academiadb"
  # Puerto en el que escuchará el servidor Backend Node.js (como string)
  PORT: "3000"
```

Aplicar y verificar el ConfigMap:
```bash
kubectl apply -f 01-configmap.yaml
kubectl get configmap app-config -o yaml
```

---

### 1.2 Creación del Secret `generic` (`db-secret`) con Base64

> **⚠️ Importante:** Es fundamental usar el flag `-n` en `echo -n` para evitar codificar un salto de línea (`\n`) al final de la credencial, lo cual provocaría fallos de autenticación en MongoDB.

#### 1. Codificar las credenciales en Base64 desde la terminal

**En Linux / macOS / Git Bash (WSL):**
```bash
echo -n 'admin_user' | base64
# Salida: YWRtaW5fdXNlcg==

echo -n 'S3cr3tP@ss2026' | base64
# Salida: UzNjcjN0UEBzczIwMjY=
```

**En Windows PowerShell:**
```powershell
[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("admin_user"))
# Salida: YWRtaW5fdXNlcg==

[Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("S3cr3tP@ss2026"))
# Salida: UzNjcjN0UEBzczIwMjY=
```

#### 2. Definir el Manifiesto del Secret (`01-secret.yaml`)

Archivo `01-secret.yaml`:
```yaml
# ==============================================================================
# ARCHIVO: 01-secret.yaml
# DESCRIPCIÓN: Almacena credenciales sensibles codificadas en Base64
# ==============================================================================
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  labels:
    app: mongodb
    tier: security
# Opaque (generic) es el tipo por defecto para pares clave-valor arbitrarios
type: Opaque
data:
  # Usuario administrador de MongoDB en Base64 (valor original: admin_user)
  DB_USER: YWRtaW5fdXNlcg==
  # Contraseña de MongoDB en Base64 (valor original: S3cr3tP@ss2026)
  DB_PASSWORD: UzNjcjN0UEBzczIwMjY=
```

*(Nota: También es posible crearlo por CLI mediante `kubectl create secret generic db-secret --from-literal=DB_USER=admin_user --from-literal=DB_PASSWORD='S3cr3tP@ss2026'`).*

#### 3. Aplicar y verificar el Secret
```bash
kubectl apply -f 01-secret.yaml
kubectl get secret db-secret
kubectl describe secret db-secret
```

---

## Paso 2: Despliegue de la Base de Datos (MongoDB)

La capa de datos requiere:
1. **`PersistentVolumeClaim` (`mongo-pvc`)**: Para que los datos de `/data/db` sobrevivan al reinicio o recreación del Pod de MongoDB.
2. **`Deployment` (`mongodb-deployment`)**: Configura MongoDB inicializando su usuario root a partir del `Secret` `db-secret` y la base de datos inicial desde el `ConfigMap` `app-config`.
3. **`Service` (`mongo-service`) de tipo `ClusterIP`**: Expone el puerto `27017` **únicamente dentro del clúster**, impidiendo el acceso directo desde el exterior.

### Manifiesto `02-mongodb.yaml`

```yaml
# ==============================================================================
# ARCHIVO: 02-mongodb.yaml
# DESCRIPCIÓN: PVC (Persistencia), Deployment y Service (ClusterIP) para MongoDB
# ==============================================================================

# 1. PersistentVolumeClaim: Solicita almacenamiento persistente a Minikube
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
  labels:
    app: mongodb
spec:
  accessModes:
    - ReadWriteOnce          # El volumen puede ser montado como lectura/escritura por un solo nodo
  resources:
    requests:
      storage: 1Gi           # Capacidad solicitada para los datos de MongoDB
---
# 2. Deployment: Gestiona el Pod de la base de datos MongoDB
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
        tier: database
    spec:
      containers:
        - name: mongodb
          image: mongo:7.0
          ports:
            - containerPort: 27017
              name: mongo-port
          # Inyección de credenciales desde el Secret y ConfigMap para inicializar MongoDB
          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_USER
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD
            - name: MONGO_INITDB_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: DB_NAME
          # Montaje del volumen persistente en el directorio de datos oficial de Mongo
          volumeMounts:
            - name: mongo-storage
              mountPath: /data/db
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: mongo-storage
          persistentVolumeClaim:
            claimName: mongo-pvc
---
# 3. Service (ClusterIP): Expone MongoDB ÚNICAMENTE dentro del clúster
apiVersion: v1
kind: Service
metadata:
  name: mongo-service
  labels:
    app: mongodb
spec:
  type: ClusterIP            # Valor por defecto: aísla la BD del tráfico externo
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017            # Puerto del Service dentro del clúster
      targetPort: 27017      # Puerto del contenedor MongoDB
```

### Comandos para desplegar y verificar MongoDB

```bash
# Aplicar el manifiesto de base de datos
kubectl apply -f 02-mongodb.yaml

# Verificar que el PVC esté en estado Bound (vinculado al StorageClass de Minikube)
kubectl get pvc mongo-pvc

# Esperar a que el Pod de MongoDB esté en estado Running (1/1)
kubectl rollout status deployment/mongodb-deployment
kubectl get pods -l app=mongodb
kubectl get svc mongo-service
```

---

## Paso 3: Despliegue de la Aplicación Backend (Node.js)

En este paso desplegaremos un Backend en **Node.js** con 2 réplicas que demuestra los dos mecanismos de inyección de variables en Kubernetes:
1. **`envFrom`**: Inyecta de forma masiva todos los pares clave-valor del `ConfigMap` `app-config` (`DB_HOST`, `DB_NAME`, `PORT`).
2. **`valueFrom` (`secretKeyRef`)**: Inyecta de forma granular las claves `DB_USER` y `DB_PASSWORD` desde el `Secret` `db-secret`.

### Manifiesto `03-backend.yaml`

```yaml
# ==============================================================================
# ARCHIVO: 03-backend.yaml
# DESCRIPCIÓN: Deployment del Backend (Node.js) inyectando ConfigMap y Secret
#              mediante envFrom (carga masiva) y valueFrom (carga selectiva).
# ==============================================================================
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  labels:
    app: backend
    tier: api
spec:
  replicas: 2                # Alta disponibilidad con 2 réplicas del Backend
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      containers:
        - name: backend-nodejs
          image: node:20-alpine
          workingDir: /app
          # Instala el driver oficial de MongoDB e inicia la API HTTP usando solo variables de entorno
          command: ["/bin/sh", "-c"]
          args:
            - |
              npm init -y >/dev/null 2>&1 && npm install mongodb >/dev/null 2>&1
              cat << 'EOF' > server.js
              const http = require('http');
              const { MongoClient } = require('mongodb');

              // Lectura exclusiva desde variables de entorno (CERO credenciales hardcodeadas)
              const PORT = process.env.PORT || 3000;
              const DB_HOST = process.env.DB_HOST;
              const DB_NAME = process.env.DB_NAME;
              const DB_USER = process.env.DB_USER;
              const DB_PASSWORD = process.env.DB_PASSWORD;

              const uri = `mongodb://${encodeURIComponent(DB_USER)}:${encodeURIComponent(DB_PASSWORD)}@${DB_HOST}:27017/${DB_NAME}?authSource=admin`;
              const client = new MongoClient(uri, { serverSelectionTimeoutMS: 3000 });

              const server = http.createServer(async (req, res) => {
                res.setHeader('Content-Type', 'application/json; charset=utf-8');
                try {
                  await client.connect();
                  await client.db(DB_NAME).command({ ping: 1 });
                  res.writeHead(200);
                  res.end(JSON.stringify({
                    status: "OK",
                    mensaje: "Conexión exitosa a MongoDB sin credenciales hardcodeadas",
                    pod: process.env.HOSTNAME,
                    configuracion_configmap: {
                      DB_HOST,
                      DB_NAME,
                      PORT
                    },
                    credenciales_secret: {
                      DB_USER,
                      DB_PASSWORD_CARGADA: Boolean(DB_PASSWORD)
                    }
                  }, null, 2));
                } catch (error) {
                  res.writeHead(500);
                  res.end(JSON.stringify({
                    status: "ERROR",
                    mensaje: "Fallo al conectar con MongoDB",
                    detalle: error.message
                  }, null, 2));
                }
              });

              server.listen(PORT, () => {
                console.log(`[BACKEND] Servidor escuchando en puerto ${PORT}`);
                console.log(`[CONFIG] Conectando a host=${DB_HOST} | db=${DB_NAME} | user=${DB_USER}`);
              });
              EOF
              node server.js
          ports:
            - containerPort: 3000
              name: http-port

          # MÉTODO 1: envFrom -> Inyecta TODAS las claves del ConfigMap como variables de entorno
          # Carga automáticamente: DB_HOST, DB_NAME y PORT
          envFrom:
            - configMapRef:
                name: app-config

          # MÉTODO 2: valueFrom -> Inyecta claves específicas desde el Secret (o ConfigMap)
          env:
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_USER
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: DB_PASSWORD

          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "300m"
```

### Comandos para desplegar y verificar el Backend

```bash
# Aplicar el Deployment del Backend
kubectl apply -f 03-backend.yaml

# Monitorear el despliegue de las 2 réplicas
kubectl rollout status deployment/backend-deployment

# Listar los Pods del Backend
kubectl get pods -l app=backend -o wide
```

---

## Paso 4: Exposición y Pruebas

Para consumir el Backend desde fuera del clúster local, crearemos un `Service` de tipo **`NodePort`**, que asigna el puerto `30080` en el nodo de Minikube y balancea la carga entre las 2 réplicas del Backend.

### Manifiesto `04-backend-service.yaml`

```yaml
# ==============================================================================
# ARCHIVO: 04-backend-service.yaml
# DESCRIPCIÓN: Expone el Backend hacia el exterior del clúster mediante NodePort
# ==============================================================================
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  labels:
    app: backend
spec:
  type: NodePort             # Abre un puerto estático (30000-32767) en la IP del nodo Minikube
  selector:
    app: backend             # Enruta el tráfico hacia los Pods con label app=backend
  ports:
    - protocol: TCP
      port: 3000             # Puerto interno del Service (ClusterIP)
      targetPort: 3000       # Puerto en el que escucha el contenedor Node.js
      nodePort: 30080        # Puerto externo fijo expuesto en el nodo de Minikube
```

### Comandos de Exposición y Prueba

```bash
# 1. Aplicar el Service NodePort
kubectl apply -f 04-backend-service.yaml

# 2. Verificar los Services del clúster (ClusterIP vs NodePort)
kubectl get svc

# 3. Obtener la URL de acceso expuesta por Minikube
minikube service backend-service --url
```

> **💡 Nota para usuarios de Docker Driver (Windows / macOS):** Con el driver de Docker, `minikube service backend-service --url` crea un túnel local hacia el contenedor de Minikube. Mantén esa terminal abierta y usa la URL entregada en una segunda terminal o en tu navegador.

```bash
# 4. Probar el endpoint desde otra terminal usando curl (reemplaza por la URL devuelta)
curl http://127.0.0.1:<PUERTO_TUNEL>
```

**Respuesta JSON esperada:**
```json
{
  "status": "OK",
  "mensaje": "Conexión exitosa a MongoDB sin credenciales hardcodeadas",
  "pod": "backend-deployment-6b7f8c9d4-x2k9p",
  "configuracion_configmap": {
    "DB_HOST": "mongo-service",
    "DB_NAME": "academiadb",
    "PORT": "3000"
  },
  "credenciales_secret": {
    "DB_USER": "admin_user",
    "DB_PASSWORD_CARGADA": true
  }
}
```
*(Si ejecutas `curl` varias veces seguidas, observarás cómo el campo `"pod"` alterna entre las 2 réplicas del Deployment gracias al balanceo de carga de `backend-service`).*

---

## Paso 5: Validación y Troubleshooting

En un entorno real, un Ingeniero DevOps valida que las variables se hayan inyectado adecuadamente y audita los logs de los contenedores.

### 5.1 Inspeccionar las variables de entorno inyectadas dentro del Pod (`kubectl exec`)

```bash
# 1. Obtener el nombre de uno de los Pods del Backend
POD_NAME=$(kubectl get pods -l app=backend -o jsonpath="{.items[0].metadata.name}")
echo "Inspeccionando Pod: $POD_NAME"

# 2. Listar las variables inyectadas por el ConfigMap y el Secret dentro del contenedor
kubectl exec -it $POD_NAME -- printenv | grep -E "DB_|PORT"
```

*En Windows PowerShell:*
```powershell
$POD_NAME = kubectl get pods -l app=backend -o jsonpath="{.items[0].metadata.name}"
kubectl exec -it $POD_NAME -- sh -c "printenv | grep -E 'DB_|PORT'"
```

**Salida esperada (valores decodificados automáticamente por el kubelet dentro del contenedor):**
```text
PORT=3000
DB_HOST=mongo-service
DB_NAME=academiadb
DB_USER=admin_user
DB_PASSWORD=S3cr3tP@ss2026
```

### 5.2 Revisión de Logs (`kubectl logs`)

```bash
# Revisar los logs de todas las réplicas del Backend usando su label
kubectl logs -l app=backend

# Seguir los logs en tiempo real (streaming) de un Pod específico
kubectl logs -f $POD_NAME

# Revisar los logs de inicialización y conexiones entrantes en MongoDB
kubectl logs -l app=mongodb --tail=30
```

### 5.3 Guía Rápida de Troubleshooting de Errores Comunes

```bash
# Inspeccionar eventos del Pod si queda en estado CreateContainerConfigError o Pending
kubectl describe pod $POD_NAME

# Verificar que los Endpoints de ambos Services apuntan a las IPs de los Pods activos
kubectl get endpoints mongo-service backend-service
```

---

## 🧹 Limpieza del Laboratorio (Opcional)

```bash
kubectl delete -f 04-backend-service.yaml -f 03-backend.yaml -f 02-mongodb.yaml -f 01-secret.yaml -f 01-configmap.yaml
```

---

## ❓ Preguntas de Validación del Aprendizaje

1. **Diferencia conceptual y de seguridad (`ConfigMap` vs `Secret`):**  
   ¿Por qué las credenciales (`DB_USER`, `DB_PASSWORD`) deben almacenarse en un `Secret` y no en un `ConfigMap`? ¿La codificación en `Base64` de un `Secret` equivale a cifrado criptográfico? ¿Qué medidas adicionales aplicarías en un clúster de producción?
2. **Inyección de configuración (`envFrom` vs `valueFrom`):**  
   En `03-backend.yaml` utilizamos `envFrom` para el `ConfigMap` y `valueFrom` (`secretKeyRef`) para el `Secret`. ¿En qué escenario elegirías uno sobre el otro?
3. **Aislamiento de red (`ClusterIP` vs `NodePort`):**  
   ¿Por qué `mongo-service` fue configurado con tipo `ClusterIP` mientras que `backend-service` utilizó `NodePort`? ¿Qué riesgo de seguridad implicaría exponer MongoDB con un `NodePort`?
4. **Actualización en caliente (Hot Reload) de variables de entorno:**  
   Si modificas el valor de `DB_NAME` en el `ConfigMap` (`kubectl edit configmap app-config`), ¿las variables de entorno dentro de los Pods en ejecución del Backend se actualizan automáticamente? ¿Qué comando debes ejecutar para que los Pods tomen el nuevo valor?
5. **Persistencia de Datos (`PVC`):**  
   Si eliminas el Pod de MongoDB con `kubectl delete pod -l app=mongodb`, Kubernetes creará uno nuevo automáticamente. ¿Qué sucede con los datos almacenados en `/data/db` y por qué?
