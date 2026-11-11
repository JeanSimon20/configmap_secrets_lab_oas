# S08 | AP6: Exposición de Servicios y Configuración Segura

**Tema:** Gestión de ConfigMaps, Secrets y exposición de Pods en Kubernetes (Minikube)  
**Requisitos previos:** Docker activo, Minikube y `kubectl` configurados.

---

## Introducción: ¿Por qué separar la configuración del código?

Un error común al iniciar en el desarrollo de aplicaciones es definir parámetros de conexión y contraseñas directamente dentro del código fuente (*hardcoding*):

```javascript
// Mala práctica: credenciales fijas en el código
const DB_HOST = "192.168.1.50";
const DB_USER = "admin";
const DB_PASS = "SuperClaveSegura2026";
```

Subir este código a un repositorio en Git expone las credenciales y obliga a reconstruir la imagen Docker cada vez que cambia un parámetro (por ejemplo, al pasar de desarrollo a producción).

En Kubernetes desacoplamos estos valores utilizando dos recursos nativos:
- **ConfigMap:** Para variables de configuración pública o no sensible (puertos, nombres de host, entorno).
- **Secret:** Para datos sensibles (usuarios, contraseñas, tokens, llaves de API).

```javascript
// Buena práctica: lectura desde variables de entorno inyectadas por Kubernetes
const DB_HOST = process.env.DB_HOST;
const DB_USER = process.env.DB_USER;
const DB_PASS = process.env.DB_PASS;
```

---

## Esquema del Laboratorio

```text
Namespace: demo-config
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  [ConfigMap: app-config]          [Secret: db-credentials]          │
│   - APP_PORT=8080                  - DB_USER (Base64)               │
│   - DB_HOST=mongo-service          - DB_PASS (Base64)               │
│   - ENVIRONMENT=production                                          │
│              │                                │                     │
│              └───────────► envFrom ◄──────────┘                     │
│                               │                                     │
│                               ▼                                     │
│                    [Pod: backend-demo-pod]                          │
│                               ▲                                     │
│                               │                                     │
│                 [Service: backend-demo-svc (NodePort)]              │
└───────────────────────────────┼─────────────────────────────────────┘
                                │
                        Acceso desde navegador
```

---

## Paso 1: Verificación de Minikube y creación del Namespace

Antes de desplegar los recursos, verificamos que el clúster local esté operativo y creamos un *Namespace* llamado `demo-config` para trabajar de forma aislada sin mezclar nuestros objetos con el namespace `default`.

### Comandos

```bash
# 1. Verificar el estado de Minikube
minikube status

# 2. Comprobar que el nodo esté en estado Ready
kubectl get nodes

# 3. Crear el namespace para la práctica
kubectl create namespace demo-config

# 4. Listar los namespaces disponibles
kubectl get namespaces
```

### Salida esperada

```text
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured

NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   15d   v1.31.0

namespace/demo-config created
```

---

## Paso 2: Creación de un ConfigMap

Crearemos el ConfigMap `app-config` desde la línea de comandos (modo imperativo) con tres variables no sensibles: `APP_PORT`, `DB_HOST` y `ENVIRONMENT`. *(En el repositorio también se incluye el archivo `configmap.yaml` como referencia de su estructura declarativa).*

### Comandos

```bash
# 1. Crear el ConfigMap en el namespace demo-config
kubectl create configmap app-config \
  --from-literal=APP_PORT=8080 \
  --from-literal=DB_HOST=mongo-service \
  --from-literal=ENVIRONMENT=production \
  -n demo-config

# 2. Verificar la creación del recurso
kubectl get configmap -n demo-config

# 3. Inspeccionar el detalle y los valores almacenados
kubectl describe configmap app-config -n demo-config
```

### Salida esperada

```text
Name:         app-config
Namespace:    demo-config
Labels:       <none>
Annotations:  <none>

Data
====
APP_PORT:
----
8080
DB_HOST:
----
mongo-service
ENVIRONMENT:
----
production
```

> **Observación:** Al ejecutar `kubectl describe`, todos los valores de un ConfigMap se muestran directamente en texto plano. Por este motivo nunca debe utilizarse para almacenar contraseñas.

---

## Paso 3: Creación de un Secret e inspección en Base64

Ahora crearemos el Secret `db-credentials` de tipo `generic` para almacenar el usuario (`DB_USER`) y la contraseña (`DB_PASS`). Luego compararemos cómo se visualiza con `describe` frente a su definición en formato YAML.

### Comandos

```bash
# 1. Crear el Secret de forma imperativa
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=SuperClaveSegura2026 \
  -n demo-config

# 2. Inspeccionar el Secret con describe
kubectl describe secret db-credentials -n demo-config

# 3. Consultar el manifiesto en YAML para ver los valores en Base64
kubectl get secret db-credentials -n demo-config -o yaml

# 4. Decodificar el valor de DB_PASS desde la terminal
echo "U3VwZXJDbGF2ZVNlZ3VyYTIwMjY=" | base64 --decode ; echo ""
```

*(Si utilizas PowerShell en Windows, el comando equivalente para decodificar es: `[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("U3VwZXJDbGF2ZVNlZ3VyYTIwMjY="))`).*

### Salida esperada

Con `kubectl describe`:
```text
Name:         db-credentials
Namespace:    demo-config
Type:         Opaque

Data
====
DB_PASS:  20 bytes
DB_USER:  5 bytes
```

Con `kubectl get secret -o yaml`:
```yaml
apiVersion: v1
data:
  DB_PASS: U3VwZXJDbGF2ZVNlZ3VyYTIwMjY=
  DB_USER: YWRtaW4=
kind: Secret
metadata:
  name: db-credentials
  namespace: demo-config
type: Opaque
```

> **Importante:** Como se comprueba al decodificar la cadena, **Base64 no es un algoritmo de encriptación**, sino una codificación en texto. En entornos reales, la seguridad de los Secrets se complementa restringiendo permisos de lectura con RBAC y habilitando el cifrado en disco (*etcd*).

---

## Paso 4: Despliegue del Pod inyectando ConfigMap y Secret (`envFrom`)

Revisaremos el archivo `pod-demo.yaml`, el cual levanta un contenedor ligero de Nginx y utiliza la propiedad `envFrom` para cargar automáticamente todas las claves de `app-config` y `db-credentials` como variables de entorno.

### Comandos

```bash
# 1. Desplegar el Pod en el clúster
kubectl apply -f pod-demo.yaml

# 2. Verificar que el Pod se encuentre en estado Running
kubectl get pods -n demo-config
```

### Salida esperada

```text
pod/backend-demo-pod created

NAME               READY   STATUS    RESTARTS   AGE
backend-demo-pod   1/1     Running   0          6s
```

---

## Paso 5: Validación en consola y exposición del servicio

### 5.1 Verificación interna mediante `kubectl exec`

Ingresamos a la terminal del contenedor para comprobar que las variables fueron inyectadas en memoria y que Kubernetes decodificó automáticamente los valores del Secret.

```bash
# 1. Ingresar a la shell del contenedor
kubectl exec -it backend-demo-pod -n demo-config -- sh

# 2. Dentro del contenedor, listar las variables inyectadas
printenv | grep -E "APP_PORT|DB_HOST|ENVIRONMENT|DB_USER|DB_PASS"

# 3. Probar la lectura de las variables
echo "Conectado a $DB_HOST ($ENVIRONMENT) con el usuario $DB_USER"

# 4. Salir del contenedor
exit
```

**Salida esperada:**
```text
APP_PORT=8080
ENVIRONMENT=production
DB_HOST=mongo-service
DB_USER=admin
DB_PASS=SuperClaveSegura2026
Conectado a mongo-service (production) con el usuario admin
```

### 5.2 Exposición del Pod con un Service `NodePort`

Para visualizar la página de estado que genera el contenedor desde nuestro navegador, exponemos el Pod a través del archivo `service-demo.yaml`.

```bash
# 1. Crear el Service de tipo NodePort
kubectl apply -f service-demo.yaml

# 2. Verificar el puerto asignado al servicio
kubectl get svc -n demo-config

# 3. Abrir el servicio en el navegador a través de Minikube
minikube service backend-demo-svc -n demo-config
```

---

## Paso 6: Prueba de actualización de variables en ejecución

¿Qué ocurre si modificamos un valor en el ConfigMap mientras el Pod ya está corriendo? Realizaremos la prueba cambiando `ENVIRONMENT` de `production` a `staging`.

```bash
# 1. Recrear el ConfigMap cambiando ENVIRONMENT a staging
kubectl delete configmap app-config -n demo-config
kubectl create configmap app-config \
  --from-literal=APP_PORT=8080 \
  --from-literal=DB_HOST=mongo-service \
  --from-literal=ENVIRONMENT=staging \
  -n demo-config

# 2. Consultar el valor actual dentro del Pod
kubectl exec backend-demo-pod -n demo-config -- printenv ENVIRONMENT
```

**Resultado:**
```text
production
```

> **Conclusión:** Las variables de entorno de un contenedor se asignan únicamente al momento de su arranque. Si se actualiza un ConfigMap o un Secret posteriormente, es necesario recrear el Pod (o reiniciar el Deployment con `kubectl rollout restart`) para que tome los nuevos valores.

---

## Ejercicios Prácticos en Clase

Antes de eliminar el entorno, resuelve las siguientes actividades en tu clúster local:

1. **Ejercicio 1 (ConfigMap):**  
   Agrega una variable adicional llamada `AUTOR_LAB` con tu nombre al ConfigMap `app-config` (puedes editar `configmap.yaml` y aplicar los cambios con `kubectl apply -f configmap.yaml`). Luego recrea el Pod y verifica con `kubectl exec` que la variable aparezca dentro del contenedor.

2. **Ejercicio 2 (Secret en Base64):**  
   Codifica en Base64 el texto `TokenLaboratorio2026` desde tu terminal. Agrega una nueva clave `API_KEY` con ese valor en el archivo `secret.yaml`, aplícalo al clúster, recrea el Pod y comprueba que al hacer `printenv` dentro del contenedor el token se muestre decodificado.

3. **Ejercicio 3 (Diagnóstico de errores):**  
   Edita `pod-demo.yaml` y cambia temporalmente el nombre del Secret `db-credentials` por `secret-inexistente`. Elimina y vuelve a aplicar el Pod.
   - ¿Qué estado muestra la columna `STATUS` al ejecutar `kubectl get pods -n demo-config`?
   - Ejecuta `kubectl describe pod backend-demo-pod -n demo-config` e identifica en la sección `Events` el mensaje exacto del error. *(Finalmente, restaura el nombre correcto).*

---

## Paso 7: Limpieza del entorno

Al haber trabajado dentro de un Namespace dedicado, podemos eliminar todos los recursos creados durante la sesión con un solo comando:

```bash
# Eliminar el namespace y todos los objetos que contiene
kubectl delete namespace demo-config

# Verificar que el namespace fue eliminado
kubectl get namespaces
```

---

## Preguntas de Repaso

1. **¿Qué diferencias existen entre un `ConfigMap` y un `Secret` al inspeccionarlos desde `kubectl`?**
2. **¿Por qué almacenar datos en `Base64` dentro de un `Secret` no equivale a tenerlos encriptados?**
3. **¿Necesita nuestra aplicación (Node.js, Java, Python) incluir alguna librería para decodificar el `Base64` cuando consume un `Secret` mediante variables de entorno? ¿Por qué?**
4. **¿Por qué un Pod en ejecución no refleja inmediatamente los cambios realizados sobre las variables de entorno de un `ConfigMap`?**
