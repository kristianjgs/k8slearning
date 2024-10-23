En Kubernetes, los manifiestos son archivos de configuración escritos en YAML o JSON que describen el estado deseado de un recurso. A continuación, te presento los tipos más comunes de manifiestos que puedes crear, una breve descripción y un ejemplo de cada uno:

### 1. **Pod**
El **Pod** es la unidad básica de ejecución en Kubernetes y puede contener uno o varios contenedores que comparten los mismos recursos.

**Descripción**: Un Pod agrupa contenedores que necesitan trabajar juntos. Los contenedores dentro de un mismo Pod comparten el mismo espacio de red y de almacenamiento.

**Ejemplo de manifiesto**:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:1.19
      ports:
        - containerPort: 80
```

---

### 2. **Deployment**
Un **Deployment** gestiona la creación y el escalado de Pods y garantiza la disponibilidad continua de las aplicaciones.

**Descripción**: Es una abstracción sobre los Pods y permite manejar réplicas de ellos para garantizar alta disponibilidad y actualización de manera controlada.

**Ejemplo de manifiesto**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.19
          ports:
            - containerPort: 80
```

---

### 3. **Service**
Un **Service** expone uno o más Pods a través de una única IP, proporcionando un punto de acceso estable.

**Descripción**: Los Services permiten la comunicación entre Pods y otros servicios dentro o fuera del clúster mediante la asignación de una IP estable y un nombre de DNS.

**Ejemplo de manifiesto**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

---

### 4. **ConfigMap**
Un **ConfigMap** se utiliza para almacenar configuraciones que pueden ser inyectadas en los Pods como variables de entorno o archivos de configuración.

**Descripción**: Permite externalizar la configuración de una aplicación, haciendo que el código y la configuración sean independientes.

**Ejemplo de manifiesto**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app_name: "MyApp"
  log_level: "INFO"
```

---

### 5. **Secret**
Un **Secret** es similar a un ConfigMap, pero está diseñado específicamente para almacenar información sensible como contraseñas o tokens.

**Descripción**: Se utiliza para gestionar secretos de forma segura, ya que permite inyectar información sensible en los Pods sin tener que codificarla en el manifiesto.

**Ejemplo de manifiesto**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  username: YWRtaW4= # valor codificado en base64
  password: MWYyZDFlMmU2N2Rm # valor codificado en base64
```

---

### 6. **PersistentVolume (PV)**
Un **PersistentVolume** es una unidad de almacenamiento en el clúster que se puede aprovisionar de manera dinámica o estática.

**Descripción**: Proporciona almacenamiento persistente para los Pods, lo que significa que los datos se mantienen aunque el Pod sea destruido y recreado.

**Ejemplo de manifiesto**:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

---

### 7. **PersistentVolumeClaim (PVC)**
Un **PersistentVolumeClaim** es una solicitud de almacenamiento de un Pod. Vincula un PersistentVolume a un Pod.

**Descripción**: Un PVC permite a los Pods reclamar almacenamiento sin tener que preocuparse por cómo o dónde se aprovisiona el volumen.

**Ejemplo de manifiesto**:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

---

### 8. **Ingress**
El **Ingress** es un recurso que proporciona reglas para enrutar el tráfico HTTP y HTTPS externo hacia los servicios de un clúster.

**Descripción**: Ofrece un punto de entrada más avanzado y flexible que los servicios, permitiendo funcionalidades como el balanceo de carga, terminación SSL y rutas basadas en la URL.

**Ejemplo de manifiesto**:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx-service
                port:
                  number: 80
```

---

### 9. **DaemonSet**
Un **DaemonSet** garantiza que un Pod específico se ejecute en todos (o en un subconjunto) de los nodos del clúster.

**Descripción**: Se utiliza para implementar tareas que deben ejecutarse en cada nodo del clúster, como agentes de monitoreo o de almacenamiento.

**Ejemplo de manifiesto**:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: daemonset-example
spec:
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: daemon-container
          image: myapp:latest
```

---

### 10. **StatefulSet**
Un **StatefulSet** gestiona el despliegue y el escalado de un conjunto de Pods con identidad persistente.

**Descripción**: Es útil para aplicaciones que requieren un almacenamiento persistente, identificación estable y orden en el despliegue y actualización de Pods, como bases de datos.

**Ejemplo de manifiesto**:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: statefulset-example
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.19
          ports:
            - containerPort: 80
```

---

### 11. **Job**
Un **Job** crea uno o más Pods y asegura que un número específico de ellos terminen con éxito. Se utiliza para tareas temporales o de ejecución única.

**Descripción**: Ejecuta un conjunto de Pods hasta que completan su tarea, garantizando que un número específico de intentos tengan éxito.

**Ejemplo de manifiesto**:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-example
spec:
  template:
    spec:
      containers:
        - name: job-container
          image: busybox
          command: ["echo", "Hello Kubernetes!"]
      restartPolicy: Never
```

---

### 12. **CronJob**
Un **CronJob** es similar a un Job, pero se ejecuta de manera programada basado en un cron.

**Descripción**: Permite ejecutar Jobs de forma recurrente en horarios o intervalos predefinidos.

**Ejemplo de manifiesto**:
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cronjob-example
spec:
  schedule: "*/5 * * * *"  # Se ejecuta cada 5 minutos
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cronjob-container
              image: busybox
              command: ["date"]
          restartPolicy: OnFailure
```

Estos son los tipos más comunes de manifiestos que puedes crear en Kubernetes. Cada uno de ellos tiene su propio propósito y puedes combinarlos para desplegar aplicaciones complejas.