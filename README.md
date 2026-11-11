# 🎓 Guía de Live-Coding para el Docente (Edición 10/10)
## S08 | AP6: Exposición de Servicios y Configuración Segura (ConfigMaps y Secrets en Minikube)

- **Audiencia:** Alumnos de nivel Junior (Introducción a Kubernetes / DevOps)
- **Modalidad:** Demostración práctica en vivo (*Live-Coding* sincrónico) + Micro-Retos Gamificados
- **Duración estimada:** 35 – 45 minutos
- **Entorno:** Minikube + `kubectl`
- **Objetivo de la Demostración:** Mostrar en vivo cómo desacoplar la configuración pública (`ConfigMap`) y las credenciales sensibles (`Secret`) de una aplicación en Kubernetes sin hardcodear datos en el código, validando el resultado tanto por terminal (`kubectl exec`) como en el navegador exponiendo el servicio con `NodePort`.

---

## 🔥 0. Gancho Inicial (3 min): El "Pecado Capital" que motiva esta clase

### 🎙️ Lo que dice el profesor (Storytelling antes de tocar la terminal)
> *"Antes de escribir nuestro primer comando, quiero contarles un caso real que le pasa a muchos desarrolladores Junior en su primer trabajo. Crean su aplicación, funciona en su laptop, hacen `git push` a un repositorio de GitHub y se van a dormir. Pero dejaron una línea como esta en su código..."*

### 👀 Lo que muestras en pantalla (El Antes vs. El Después)

```javascript
// ❌ EL ANTI-PATRÓN (Hardcoding: Credenciales pegadas en el código fuente)
const DB_HOST = "192.168.1.50";
const DB_USER = "admin";
const DB_PASS = "SuperClaveSegura2026"; // <-- Si subes esto a Git, bots lo roban en segundos
```

```javascript
// ✅ EL PATRÓN CLOUD-NATIVE (12-Factor App: El código solo lee variables de entorno)
const DB_HOST = process.env.DB_HOST; // Viene de un ConfigMap en Kubernetes
const DB_USER = process.env.DB_USER; // Viene de un Secret en Kubernetes
const DB_PASS = process.env.DB_PASS; // Viene de un Secret en Kubernetes
```

> *"Si un cliente te pide cambiar el puerto o rotar la contraseña de la base de datos y la tienes 'hardcodeada', tienes que modificar el código, volver a construir la imagen Docker y volver a subirla. Con **ConfigMaps** y **Secrets**, la imagen Docker nunca se toca: Kubernetes le inyecta los datos desde afuera cuando el Pod arranca. ¡Vamos a construirlo en vivo!"*

---

## 🗺️ Mapa Visual de lo que Construiremos Hoy

```text
┌─────────────────────────────────────────────────────────────────────────┐
│ Namespace: demo-config (Clúster Minikube)                               │
│                                                                         │
│  ┌──────────────────────────────┐     ┌──────────────────────────────┐  │
│  │ ConfigMap: app-config        │     │ Secret: db-credentials       │  │
│  │ (Texto Plano - No sensible)  │     │ (Codificado en Base64)       │  │
│  │  • APP_PORT = 8080           │     │  • DB_USER = YWRtaW4=        │  │
│  │  • DB_HOST  = mongo-service  │     │  • DB_PASS = U3VwZXJDb...    │  │
│  │  • ENVIRONMENT = production  │     └──────────────┬───────────────┘  │
│  └──────────────┬───────────────┘                    │                  │
│                 │              envFrom               │                  │
│                 └────────────────┐  ┌────────────────┘                  │
│                                  ▼  ▼                                   │
│               ┌────────────────────────────────────────┐                │
│               │ Pod: backend-demo-pod                  │                │
│               │  • Variables en memoria (printenv)     │                │
│               │  • Dashboard Web en puerto 80          │                │
│               └────────────────────┬───────────────────┘                │
│                                    ▲                                    │
│               ┌────────────────────┴───────────────────┐                │
│               │ Service: backend-demo-svc (NodePort)   │                │
│               └────────────────────┬───────────────────┘                │
└────────────────────────────────────┼────────────────────────────────────┘
                                     ▼
                        [ Navegador del Alumno 🌐 ]
```

---

## Paso 1: Verificación de Minikube y Creación de un Namespace Aislado (`demo-config`)

### 🎙️ Lo que dice el profesor
> *"Regla número uno de un ingeniero DevOps: antes de desplegar, verificamos que el clúster esté saludable y nunca ensuciamos el namespace `default`. Vamos a revisar el estado de Minikube y crearemos un entorno aislado llamado `demo-config` (un Namespace). Es como crear una carpeta o habitación exclusiva para nuestro experimento de hoy."*

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

- **🔍 Qué señalar en pantalla:** Muestra a los alumnos que `host`, `kubelet` y `apiserver` están en `Running` y que `demo-config` aparece como `Active`.

---

## Paso 2: Creación de un ConfigMap Imperativo (`app-config`)

### 🎙️ Lo que dice el profesor
> *"Empecemos por los datos que **no son secretos**: el puerto de la app (`APP_PORT`), el nombre del servidor de base de datos (`DB_HOST`) y el entorno (`ENVIRONMENT`). Para esto usamos un **ConfigMap**. Lo crearemos directamente con un comando (modo imperativo) y luego lo inspeccionaremos por dentro."*

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

*(Nota: En el repositorio tienen el archivo `configmap.yaml` como referencia de cómo se ve este mismo objeto en formato YAML).*

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

- **🔍 Qué señalar en pantalla:** Destaca la columna `DATA: 3` y haz notar que al ejecutar `kubectl describe`, **todos los valores se leen abiertamente en texto plano**. Por eso una contraseña jamás debe ir aquí.

---

## Paso 3: Creación de un Secret `generic` Imperativo e Inspección del Base64

### 🎙️ Lo que dice el profesor
> *"Ahora guardaremos los datos sensibles: `DB_USER` y `DB_PASS`. Para esto usamos un **Secret**. Quiero que presten mucha atención a dos cosas: primero, cómo `kubectl describe` protege nuestra pantalla para que nadie detrás de nosotros vea la clave; y segundo, qué aparece cuando pedimos el archivo YAML (`-o yaml`). Veremos texto en **Base64**. Pero cuidado: muchos creen que Base64 significa que está encriptado. ¡Veamos en vivo si eso es cierto!"*

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

# 4. Demostración en vivo: Decodificar el Base64 de DB_PASS en un segundo
echo "U3VwZXJDbGF2ZVNlZ3VyYTIwMjY=" | base64 --decode ; echo ""
```

*(En Windows PowerShell para el punto 4: `[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String("U3VwZXJDbGF2ZVNlZ3VyYTIwMjY="))`).*

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
  1. `kubectl describe` oculta los valores mostrando únicamente `20 bytes` y `5 bytes`.
  2. Pero si alguien tiene permisos para hacer `get secret -o yaml`, puede copiar `U3VwZXJDbGF2ZVNlZ3VyYTIwMjY=`, pasarlo por `base64 --decode` y ver `SuperClaveSegura2026` sin ninguna llave. **Lección clave: Base64 es codificación, NO encriptación.**

---

## Paso 4: Creación y Despliegue de un Pod Consumiendo Ambas Fuentes (`envFrom`)

### 🎙️ Lo que dice el profesor
> *"Hasta ahora el ConfigMap y el Secret son como dos cajas guardadas en el almacén de Kubernetes; ningún contenedor las está usando todavía. Ahora vamos a desplegar nuestro Pod (`pod-demo.yaml`). Fíjense en la sección `envFrom` al final del archivo: con solo dos bloques (`configMapRef` y `secretRef`), Kubernetes inyectará automáticamente las 5 variables dentro del contenedor al arrancar."*

### 💻 Comandos a ejecutar

```bash
# 1. Aplicar el manifiesto del Pod (que incluye el Dashboard visual)
kubectl apply -f pod-demo.yaml

# 2. Verificar que el Pod esté en estado Running (1/1)
kubectl get pods -n demo-config
```

### 🖥️ Resultado esperado en pantalla (Explicación visual)

```text
pod/backend-demo-pod created

NAME               READY   STATUS    RESTARTS   AGE
backend-demo-pod   1/1     Running   0          5s
```

- **🔍 Qué señalar en pantalla:** Abre el archivo `pod-demo.yaml` en el editor, señala las últimas 8 líneas (`envFrom`) y muestra que en la terminal el Pod ya está `1/1 Running`.

---

## Paso 5: Validación Doble en Vivo (Terminal + Navegador Web)

### 5A. Validación por dentro del contenedor (`kubectl exec -it`)

#### 🎙️ Lo que dice el profesor
> *"Primero entremos a las entrañas del contenedor como lo haría un administrador de sistemas usando `kubectl exec -it`. Quiero que vean cómo llegaron las variables del Secret: ¿habrán llegado en Base64 o Kubernetes ya hizo el trabajo sucio de decodificarlas por nosotros?"*

#### 💻 Comandos a ejecutar

```bash
# 1. Abrir una terminal interactiva dentro del contenedor
kubectl exec -it backend-demo-pod -n demo-config -- sh

# --- DENTRO DEL CONTENEDOR (Prompt: / #) ---

# 2. Filtrar las 5 variables inyectadas en memoria
printenv | grep -E "APP_PORT|DB_HOST|ENVIRONMENT|DB_USER|DB_PASS"

# 3. Salir del contenedor
exit
```

#### 🖥️ Resultado esperado en pantalla

```text
/ # printenv | grep -E "APP_PORT|DB_HOST|ENVIRONMENT|DB_USER|DB_PASS"
APP_PORT=8080
ENVIRONMENT=production
DB_HOST=mongo-service
DB_USER=admin
DB_PASS=SuperClaveSegura2026
/ # exit
```

- **🔍 Qué señalar en pantalla:** Muestra que `DB_PASS` vale `SuperClaveSegura2026` en texto claro dentro de la memoria del contenedor. Tu código en Node.js, Java o Python no tiene que decodificar Base64; lo lee directamente.

---

### 5B. Exposición del Servicio (`NodePort`) y Efecto Visual en el Navegador

#### 🎙️ Lo que dice el profesor
> *"Verlo en la terminal está bien, pero nuestro tema de hoy también es **Exposición de Servicios**. Nuestro Pod vive encerrado dentro de la red privada de Kubernetes. Vamos a exponerlo hacia afuera creando un **Service de tipo `NodePort`** y abriremos el Dashboard en el navegador para ver cómo nuestra aplicación web consume el ConfigMap y el Secret en tiempo real."*

#### 💻 Comandos a ejecutar

```bash
# 1. Exponer el Pod mediante un Service tipo NodePort (o aplicar service-demo.yaml)
kubectl apply -f service-demo.yaml

# 2. Verificar que el Service tiene asignado el puerto externo 30080
kubectl get svc -n demo-config

# 3. Abrir automáticamente el servicio en el navegador con Minikube
minikube service backend-demo-svc -n demo-config
```

#### 🖥️ Resultado esperado en pantalla (Terminal y Navegador)

```text
service/backend-demo-svc created

NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
backend-demo-svc   NodePort   10.104.22.118   <none>        80:30080/TCP   4s
```

- **🔍 Qué señalar en el navegador:** Se abrirá automáticamente una tarjeta web oscura mostrando:
  - **📦 Desde ConfigMap (`app-config`):** `ENVIRONMENT: production`, `DB_HOST: mongo-service`, `APP_PORT: 8080`.
  - **🔐 Desde Secret (`db-credentials`):** `DB_USER: admin`, `DB_PASS: ✅ Cargada en memoria (20 caracteres)`.

---

## ⚡ Paso 6: El Experimento en Vivo ("Aprender Rompiendo") — El Mito del Hot-Reload

### 🎙️ Lo que dice el profesor
> *"Ahora les voy a hacer una pregunta con trampa donde cae hasta gente con experiencia. ¿Qué pasa si borro el `ConfigMap` actual, lo vuelvo a crear cambiando `ENVIRONMENT=staging`, y reviso las variables dentro del Pod que ya está corriendo? ¿Cambiará sola la variable `ENVIRONMENT` dentro del Pod o seguirá diciendo `production`? ¡Hagamos la apuesta y probémoslo!"*

### 💻 Comandos a ejecutar

```bash
# 1. Reemplazar el ConfigMap cambiando ENVIRONMENT de 'production' a 'staging'
kubectl delete configmap app-config -n demo-config
kubectl create configmap app-config \
  --from-literal=APP_PORT=8080 \
  --from-literal=DB_HOST=mongo-service \
  --from-literal=ENVIRONMENT=staging \
  -n demo-config

# 2. Consultar cuánto vale ENVIRONMENT dentro del Pod en ejecución
kubectl exec backend-demo-pod -n demo-config -- printenv ENVIRONMENT
```

### 🖥️ Resultado esperado en pantalla

```text
production
```

- **🔍 Explicación clave del profesor:**
  > *"¡Sigue diciendo `production`! ¿Por qué? Porque en Linux y en contenedores, **las variables de entorno se inyectan únicamente en el instante en que el contenedor arranca**. Si cambias un `ConfigMap` o un `Secret` después, los Pods que ya estaban vivos no se enteran hasta que se reinicien o recreen."*

---

## 🚀 Misión Relámpago para el Alumno (10 minutos antes de limpiar)

Antes de borrar el laboratorio, reta a los alumnos a resolver estos **3 desafíos progresivos** en sus propias máquinas:

### 🥉 Reto Bronce: "Deja tu huella en el Pod"
1. Agrega una cuarta variable llamada `AUTOR_LAB=<TuNombre>` al `ConfigMap` `app-config`.
2. Recrea el Pod (`kubectl delete -f pod-demo.yaml && kubectl apply -f pod-demo.yaml`).
3. Ejecuta `kubectl exec backend-demo-pod -n demo-config -- printenv AUTOR_LAB` y comparte captura de pantalla con tu nombre apareciendo desde adentro del contenedor.

<details>
<summary>👁️ Ver solución del Reto Bronce (Para el profesor)</summary>

```bash
kubectl delete configmap app-config -n demo-config
kubectl create configmap app-config \
  --from-literal=APP_PORT=8080 \
  --from-literal=DB_HOST=mongo-service \
  --from-literal=ENVIRONMENT=staging \
  --from-literal=AUTOR_LAB=JuanPerez \
  -n demo-config

kubectl delete -f pod-demo.yaml
kubectl apply -f pod-demo.yaml
kubectl exec backend-demo-pod -n demo-config -- printenv AUTOR_LAB
```
</details>

---

### 🥈 Reto Plata: "Cazador de Base64"
1. Codifica en Base64 la palabra `TokenUltraSecreto99` usando tu terminal.
2. Edita el archivo `secret.yaml` agregando la llave `API_KEY` con ese valor en Base64 y aplícalo con `kubectl apply -f secret.yaml`.
3. Recrea el Pod y verifica con `printenv` que `API_KEY` aparezca decodificada automáticamente.

<details>
<summary>👁️ Ver solución del Reto Plata (Para el profesor)</summary>

```bash
# 1. Obtener Base64 (Resultado: VG9rZW5VbHRyYVNlY3JldG85OQ==)
echo -n "TokenUltraSecreto99" | base64

# 2. Agregar API_KEY: VG9rZW5VbHRyYVNlY3JldG85OQ== en secret.yaml y aplicar:
kubectl apply -f secret.yaml
kubectl delete -f pod-demo.yaml && kubectl apply -f pod-demo.yaml
kubectl exec backend-demo-pod -n demo-config -- printenv API_KEY
```
</details>

---

### 🥇 Reto Oro (Troubleshooting Real): "Provoca y diagnostica el error clásico"
1. En `pod-demo.yaml`, cambia el nombre del Secret de `db-credentials` a `secret-fantasma` (un Secret que no existe).
2. Recrea el Pod y ejecuta `kubectl get pods -n demo-config`.
3. **Pregunta del reto:** ¿Qué estado exacto aparece en la columna `STATUS` y con qué comando puedes ver el mensaje de error detallado que explica qué recurso falta? *(Luego corrígelo a `db-credentials`).*

<details>
<summary>👁️ Ver solución del Reto Oro (Para el profesor)</summary>

- **Estado en `STATUS`:** `CreateContainerConfigError`
- **Comando para diagnosticar la causa exacta:**
  ```bash
  kubectl describe pod backend-demo-pod -n demo-config
  ```
  *(En la sección `Events` al final dirá: `Error: secret "secret-fantasma" not found`).*
</details>

---

## Paso 7: Limpieza del Entorno (`kubectl delete namespace`)

### 🎙️ Lo que dice el profesor
> *"Para cerrar nuestro laboratorio con buenas prácticas, limpiaremos nuestro clúster. Como fuimos ordenados y metimos nuestro ConfigMap, Secret, Pod y Service dentro del namespace `demo-config`, con un solo comando destruiremos toda la infraestructura de prueba."*

### 💻 Comandos a ejecutar

```bash
# 1. Eliminar el Namespace completo y todos sus recursos en cascada
kubectl delete namespace demo-config

# 2. Comprobar que el clúster quedó limpio
kubectl get namespaces
```

### 🖥️ Resultado esperado en pantalla

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

Realiza estas **3 preguntas finales** para consolidar los conceptos de la sesión:

### 1. "¿Qué diferencia real vimos hoy entre un `ConfigMap` y un `Secret` al inspeccionarlos en la terminal?"
- **Respuesta esperada:** En el `ConfigMap`, los valores se ven en texto plano al hacer `kubectl describe` o ver el YAML. En el `Secret`, `kubectl describe` oculta los valores (mostrando solo su tamaño en bytes) y el YAML los almacena codificados en `Base64`.
- **💡 Refuerzo del profesor:** *"Además, en un clúster empresarial configuramos permisos (RBAC) distintos para que los desarrolladores puedan ver los `ConfigMaps`, pero tengan bloqueado el acceso a los `Secrets` de producción."*

### 2. "Si un `Secret` guarda las contraseñas en `Base64`, ¿significa que están encriptadas?"
- **Respuesta esperada:** No. `Base64` es solo una codificación reversible sin clave (`base64 --decode`), no es cifrado.
- **💡 Refuerzo del profesor:** *"Por eso jamás subimos un archivo `secret.yaml` con Base64 a un repositorio público de Git. En producción activamos el cifrado en disco de Kubernetes (`etcd`) o usamos bóvedas externas como Vault, AWS Secrets Manager o External Secrets."*

### 3. "Si cambio un valor en un `ConfigMap`, ¿las variables de entorno dentro de un Pod que ya está corriendo se actualizan solas?"
- **Respuesta esperada:** No, las variables de entorno de un contenedor se inyectan solo cuando el contenedor arranca y son inmutables durante su vida; para que tome el cambio es necesario recrear o reiniciar los Pods.
- **💡 Refuerzo del profesor:** *"¡Exacto! Por eso cuando trabajamos con `Deployments` en producción y actualizamos un `ConfigMap`, ejecutamos `kubectl rollout restart deployment/<nombre>` para que Kubernetes reemplace los Pods viejos por nuevos con la configuración fresca."*
