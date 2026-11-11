# 🎓 Guía de Live-Coding para el Docente
## S08 | AP6: Exposición de Servicios y Configuración Segura (ConfigMaps y Secrets en Minikube)

- **Audiencia:** Alumnos de nivel Junior (Introducción a Kubernetes / DevOps)
- **Modalidad:** Demostración práctica en vivo (*Live-Coding* sincrónico)
- **Duración estimada:** 25 – 30 minutos
- **Entorno:** Minikube + `kubectl`
- **Objetivo de la Demostración:** Mostrar en vivo cómo desacoplar la configuración pública (`ConfigMap`) y las credenciales sensibles (`Secret`) de un contenedor en Kubernetes, inyectándolas automáticamente como variables de entorno sin hardcodear ningún dato en el código ni en la imagen Docker.

---

## 🗺️ Mapa Visual de la Demostración

```
┌────────────────────────────────────────────────────────────────────┐
│ Namespace: demo-config (Clúster Minikube)                          │
│                                                                    │
│  ┌──────────────────────────────┐  ┌────────────────────────────┐  │
│  │ ConfigMap: app-config        │  │ Secret: db-credentials     │  │
│  │ (Texto Plano - No sensible)  │  │ (Codificado en Base64)     │  │
│  │  • APP_PORT = 8080           │  │  • DB_USER = YWRtaW4=      │  │
│  │  • DB_HOST  = mongo-service  │  │  • DB_PASS = U3VwZXJDb...  │  │
│  │  • ENVIRONMENT = production  │  └─────────────┬──────────────┘  │
│  └──────────────┬───────────────┘                │                 │
│                 │           envFrom              │                 │
│                 └──────────────┐  ┌──────────────┘                 │
│                                ▼  ▼                                │
│               ┌────────────────────────────────────┐               │
│               │ Pod: backend-demo-pod              │               │
│               │ Contenedor: app-container          │               │
│               │                                    │               │
│               │ Variables de Entorno en Memoria:   │               │
│               │  APP_PORT=8080                     │               │
│               │  DB_HOST=mongo-service             │               │
│               │  ENVIRONMENT=production            │               │
│               │  DB_USER=admin                     │               │
│               │  DB_PASS=SuperClaveSegura2026      │               │
│               └────────────────────────────────────┘               │
└────────────────────────────────────────────────────────────────────┘
```

---

## Paso 1: Verificación de Minikube y Creación de un Namespace Aislado (`demo-config`)

### 🎙️ Lo que dice el profesor
> *"Chicos, antes de desplegar cualquier cosa en Kubernetes, la regla número uno de un DevOps es verificar que nuestro clúster esté vivo y nunca trabajar directamente en el namespace `default` si queremos mantener el orden. Vamos a comprobar el estado de Minikube y crearemos una 'habitación privada' llamada `demo-config` (un Namespace). Así todo lo que creemos hoy vivirá aislado y al final podremos borrar el laboratorio entero con un solo comando."*

### 💻 Comandos a ejecutar

```bash
# 1. Verificar que el clúster local de Minikube está activo
minikube status

# 2. Comprobar que nuestro nodo responde en estado Ready
kubectl get nodes

# 3. Crear un Namespace aislado para nuestra demostración
kubectl create namespace demo-config

# 4. Verificar que el Namespace fue creado
kubectl get namespaces
```

### 🖥️ Resultado esperado en pantalla (Explicación visual)

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

NAME              STATUS   AGE
default           Active   15d
demo-config       Active   4s
kube-node-lease   Active   15d
kube-public       Active   15d
kube-system       Active   15d
```

- **🔍 Qué señalar en pantalla:** Muestra a los alumnos que `host`, `kubelet` y `apiserver` aparecen como `Running` y que en la lista de namespaces ahora aparece `demo-config` con estado `Active`.

---

## Paso 2: Creación de un ConfigMap Imperativo (`app-config`)

### 🎙️ Lo que dice el profesor
> *"Imaginen que nuestra aplicación necesita saber en qué puerto escuchar (`APP_PORT`), a qué servidor de base de datos conectarse (`DB_HOST`) y en qué entorno está corriendo (`ENVIRONMENT`). Si escribimos eso fijo dentro del código y mañana cambiamos de puerto, tendríamos que recompilar la imagen Docker. Para evitarlo existe el **ConfigMap**: un diccionario clave-valor pensado para configuración **pública o no sensible**. Vamos a crearlo directo desde la terminal (modo imperativo) y ver cómo luce por dentro."*

### 💻 Comandos a ejecutar

```bash
# 1. Crear el ConfigMap de forma imperativa con 3 variables públicas
kubectl create configmap app-config \
  --from-literal=APP_PORT=8080 \
  --from-literal=DB_HOST=mongo-service \
  --from-literal=ENVIRONMENT=production \
  -n demo-config

# 2. Listar los ConfigMaps del namespace demo-config
kubectl get configmap -n demo-config

# 3. Inspeccionar el contenido interno del ConfigMap
kubectl describe configmap app-config -n demo-config
```

### 🖥️ Resultado esperado en pantalla (Explicación visual)

```text
configmap/app-config created

NAME               DATA   AGE
app-config         3      8s
kube-root-ca.crt   1      45s

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

- **🔍 Qué señalar en pantalla:** Destaca la columna `DATA: 3` (las tres variables creadas) y haz notar que al ejecutar `kubectl describe`, **todos los valores se ven en texto plano** (`8080`, `mongo-service`, `production`). Por eso **jamás** debemos guardar contraseñas en un `ConfigMap`.

---

## Paso 3: Creación de un Secret `generic` Imperativo e Inspección del Base64

### 🎙️ Lo que dice el profesor
> *"Ahora viene la pregunta del millón: ¿dónde guardamos el usuario y la contraseña de la base de datos? Para eso Kubernetes tiene los **Secrets**. Vamos a crear un Secret de tipo `generic` con `DB_USER` y `DB_PASS`. Fíjense en dos cosas muy importantes: primero, qué pasa cuando hacemos un `describe` (Kubernetes protege la pantalla ocultando los valores en bytes), y segundo, qué pasa cuando pedimos el formato YAML: veremos los valores en **Base64**. ¡Mucho ojo! Base64 NO es encriptación, es solo codificación, y se los voy a demostrar en vivo."*

### 💻 Comandos a ejecutar

```bash
# 1. Crear el Secret generic de forma imperativa
kubectl create secret generic db-credentials \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASS=SuperClaveSegura2026 \
  -n demo-config

# 2. Inspeccionar el Secret con describe (observa cómo oculta los valores)
kubectl describe secret db-credentials -n demo-config

# 3. Ver el manifiesto YAML real del Secret para observar el Base64
kubectl get secret db-credentials -n demo-config -o yaml

# 4. Demostración en vivo: Decodificar el Base64 de DB_PASS en la terminal
echo "U3VwZXJDbGF2ZVNlZ3VyYTIwMjY=" | base64 --decode ; echo ""
```

### 🖥️ Resultado esperado en pantalla (Explicación visual)

**Al ejecutar `kubectl describe secret`:**
```text
Name:         db-credentials
Namespace:    demo-config
Type:         Opaque

Data
====
DB_PASS:  20 bytes
DB_USER:  5 bytes
```

**Al ejecutar `kubectl get secret -o yaml`:**
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

**Al decodificar el Base64:**
```text
SuperClaveSegura2026
```

- **🔍 Qué señalar en pantalla:**
  1. En `kubectl describe secret`, Kubernetes solo muestra `20 bytes` y `5 bytes` para que nadie que esté mirando tu pantalla (*shoulder surfing*) o los logs de auditoría básicos vea la contraseña.
  2. En `kubectl get secret -o yaml`, aparece `U3VwZXJDbGF2ZVNlZ3VyYTIwMjY=`. Al pasar ese texto por `base64 --decode`, obtenemos `SuperClaveSegura2026` sin necesitar ninguna llave. Por eso en el mundo real restringimos con permisos (**RBAC**) quién puede ejecutar `get secret`.

---

## Paso 4: Creación y Despliegue de un Pod Consumiendo Ambas Fuentes (`envFrom`)

### 🎙️ Lo que dice el profesor
> *"Ya tenemos la configuración pública en el `ConfigMap` y las credenciales en el `Secret`, pero están guardadas en el clúster sin que nadie las use. Ahora vamos a crear un Pod (`backend-demo-pod`) y le diremos a Kubernetes: 'Toma todas las llaves de `app-config` y todas las llaves de `db-credentials` e inyéctalas como variables de entorno dentro de mi contenedor'. Para no escribir variable por variable, usaremos la propiedad mágica **`envFrom`**."*

### 💻 Manifiesto a crear (`pod-demo.yaml`) y comandos a ejecutar

Archivo `pod-demo.yaml`:
```yaml
# ==============================================================================
# ARCHIVO: pod-demo.yaml
# DESCRIPCIÓN: Pod de demostración (Live-Coding) que inyecta todas las claves
#              de un ConfigMap y un Secret simultáneamente usando envFrom.
# ==============================================================================
apiVersion: v1
kind: Pod
metadata:
  name: backend-demo-pod
  namespace: demo-config
  labels:
    app: backend-demo
    role: api
spec:
  containers:
    - name: app-container
      image: nginx:alpine
      ports:
        - containerPort: 8080
      # envFrom inyecta automáticamente TODAS las claves como variables de entorno
      envFrom:
        # 1. Inyección de configuración pública (APP_PORT, DB_HOST, ENVIRONMENT)
        - configMapRef:
            name: app-config
        # 2. Inyección de credenciales sensibles (DB_USER, DB_PASS)
        - secretRef:
            name: db-credentials
```

Comandos en terminal:
```bash
# 1. Aplicar el manifiesto del Pod
kubectl apply -f pod-demo.yaml

# 2. Verificar que el Pod esté en estado Running (1/1)
kubectl get pods -n demo-config
```

### 🖥️ Resultado esperado en pantalla (Explicación visual)

```text
pod/backend-demo-pod created

NAME               READY   STATUS    RESTARTS   AGE
backend-demo-pod   1/1     Running   0          6s
```

- **🔍 Qué señalar en pantalla:** Resalta el bloque `envFrom` del archivo YAML (cómo en solo 6 líneas conectamos tanto el `ConfigMap` como el `Secret`) y muestra que el Pod pasó rápidamente a estado `Running` (`1/1`). *(Tip: menciona que si hubiésemos escrito mal el nombre del Secret o ConfigMap, el `STATUS` diría `CreateContainerConfigError`).*

---

## Paso 5: Validación en Vivo Ingresando al Contenedor (`kubectl exec -it`)

### 🎙️ Lo que dice el profesor
> *"Llegó el momento de la verdad. Nuestro Pod ya está corriendo, pero ¿de verdad tiene adentro las 5 variables? ¿Y cómo llegan las variables del Secret: en Base64 o en texto normal listo para que nuestro código Node.js, Python o Java las lea? Vamos a meternos dentro del contenedor en vivo usando `kubectl exec -it` y ejecutaremos `printenv`."*

### 💻 Comandos a ejecutar

```bash
# 1. Abrir una terminal interactiva (shell) dentro del contenedor en ejecución
kubectl exec -it backend-demo-pod -n demo-config -- sh

# --- DENTRO DEL CONTENEDOR (Prompt: / #) ---

# 2. Filtrar las variables de entorno que inyectamos
printenv | grep -E "APP_PORT|DB_HOST|ENVIRONMENT|DB_USER|DB_PASS"

# 3. Probar la lectura individual como lo haría nuestra aplicación
echo "Conectando a $DB_HOST en entorno $ENVIRONMENT con usuario $DB_USER y clave $DB_PASS"

# 4. Salir del contenedor
exit
```

### 🖥️ Resultado esperado en pantalla (Explicación visual)

```text
/ # printenv | grep -E "APP_PORT|DB_HOST|ENVIRONMENT|DB_USER|DB_PASS"
APP_PORT=8080
ENVIRONMENT=production
DB_HOST=mongo-service
DB_USER=admin
DB_PASS=SuperClaveSegura2026

/ # echo "Conectando a $DB_HOST en entorno $ENVIRONMENT con usuario $DB_USER y clave $DB_PASS"
Conectando a mongo-service en entorno production con usuario admin y clave SuperClaveSegura2026

/ # exit
```

- **🔍 Qué señalar en pantalla:**
  1. ¡Están las **5 variables** juntas en memoria dentro del contenedor, sin importar si vinieron del `ConfigMap` o del `Secret`!
  2. Observa `DB_PASS=SuperClaveSegura2026`: **Kubernetes decodificó el Base64 automáticamente** antes de entregárselo al contenedor. Así tu aplicación no tiene que preocuparse por decodificar nada; simplemente lee `process.env.DB_PASS` o `System.getenv("DB_PASS")`.

---

## Paso 6: Limpieza del Entorno (`kubectl delete namespace`)

### 🎙️ Lo que dice el profesor
> *"¿Recuerdan que en el Paso 1 creamos el Namespace `demo-config` para mantener todo ordenado? Miren la ventaja de trabajar con Namespaces: en lugar de borrar el Pod, luego el Secret y luego el ConfigMap uno por uno, borramos el Namespace completo y Kubernetes destruye en cascada todos los recursos que vivían adentro."*

### 💻 Comandos a ejecutar

```bash
# 1. Eliminar el Namespace completo y todos sus recursos asociados
kubectl delete namespace demo-config

# 2. Comprobar que el Namespace ya no existe
kubectl get namespaces
```

### 🖥️ Resultado esperado en pantalla (Explicación visual)

```text
namespace "demo-config" deleted

NAME              STATUS   AGE
default           Active   15d
kube-node-lease   Active   15d
kube-public       Active   15d
kube-system       Active   15d
```

---

## 🎯 Cierre y Preguntas Interactivas para la Clase

Al finalizar la demostración, lanza estas **3 preguntas clave** al grupo para validar la comprensión:

### Pregunta 1: "¿Qué diferencia real vieron hoy entre un `ConfigMap` y un `Secret` cuando los inspeccionamos en la terminal?"
- **Respuesta esperada del alumno:** En el `ConfigMap` los datos se almacenan y muestran directamente en texto plano al hacer `kubectl describe` o `get -o yaml`. En cambio, en el `Secret`, `kubectl describe` oculta el contenido mostrando solo el tamaño en bytes, y en el YAML los datos se guardan codificados en `Base64`.
- **💡 Refuerzo del profesor:** *"Exacto. Y recuerden que además en Kubernetes podemos aplicar permisos (RBAC) distintos para que un desarrollador pueda ver los `ConfigMaps`, pero tenga prohibido leer los `Secrets`."*

### Pregunta 2: "Si un `Secret` guarda la contraseña en `Base64`, ¿significa que la contraseña está encriptada de forma segura?"
- **Respuesta esperada del alumno:** No. `Base64` es solo una codificación (representación de texto), no es cifrado. Cualquiera que pueda ver el YAML del `Secret` puede ejecutar `base64 --decode` y obtener la contraseña original en un segundo.
- **💡 Refuerzo del profesor:** *"¡Tal cual lo vimos en el Paso 3! Para que un Secret sea realmente seguro en producción, debemos restringir el acceso con RBAC, activar el cifrado en reposo de Kubernetes (Encryption at Rest en etcd) o usar bóvedas externas como HashiCorp Vault o AWS/GCP Secret Manager."*

### Pregunta 3: "Dentro de nuestro código de aplicación (Node.js, Java, Python), ¿tenemos que programar alguna función para decodificar el `Base64` del `Secret`?"
- **Respuesta esperada del alumno:** No, porque cuando Kubernetes inyecta el `Secret` en el Pod mediante `envFrom` (o `valueFrom`), lo decodifica automáticamente y la aplicación lo recibe en texto plano como una variable de entorno normal.
- **💡 Refuerzo del profesor:** *"¡Bingo! Para el contenedor, una variable que vino de un `ConfigMap` y una que vino de un `Secret` se leen exactamente igual."*
