### **Paso 1: Instalar el NFS CSI Driver**

El **NFS CSI Driver** oficial puede ser instalado usando los manifiestos disponibles en su repositorio de GitHub.

1. Aplica los manifiestos del controlador NFS CSI:
   
   ```bash
   curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.9.0/deploy/install-driver.sh | bash -s v4.9.0 --
   ```
   
   Esto instalará:
   
   - El controlador CSI para NFS.
   - Los controladores de usuario (Pods) en los nodos.

2. Verifica que los Pods del controlador CSI estén en ejecución:
   
   ```bash
   kubectl get pods -n kube-system
   ```
   
   Debes ver algo similar a esto:
   
   ```plaintext
   NAME                           READY   STATUS    RESTARTS   AGE
   nfs-csi-controller-0           3/3     Running   0          2m
   nfs-csi-node-abcde             3/3     Running   0          2m
   ```

Consulta el [repositorio oficial de NFS CSI Driver](https://github.com/kubernetes-csi/csi-driver-nfs) para más detalles.

---

### **Paso 2: Crear un StorageClass para NFS**

Crea un archivo YAML para definir el **StorageClass**:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi-sc
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.122.110   # Dirección IP del servidor NFS
  share: /NFS/shared1       # Ruta exportada en el servidor NFS
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

Aplica el StorageClass:

```bash
kubectl apply -f storageclass-nfs.yaml
```

---

### **Paso 3: Crear un PersistentVolumeClaim (PVC)**

Define un PVC que utilice el StorageClass creado:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs-csi-sc
```

Aplica el PVC:

```bash
kubectl apply -f pvc-nfs.yaml
```

Verifica el estado del PVC:

```bash
kubectl get pvc
```

Si está funcionando correctamente, deberías ver algo como esto:

```plaintext
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
nfs-pvc    Bound    pvc-1234abcd-5678-efgh-9101-ijklmnopqr     5Gi        RWX            nfs-csi-sc      1m
```

---

### **Paso 4: Crear certificados SSL**

Para poder crear un servicio Registry se necesitan los certificados SSL. Para realizar estos se necesita tener balanceado el cluster.

Debemos primero que todo crear los certificados:

```bash
[root@master1 registry]# mkdir certs
[root@master1 registry]# openssl req -newkey rsa:4096 -nodes -sha256 -keyout ./certs/registry.key -addext "subjectAltName = OPCION" -x509 -days 3650 -out ./certs/registry.crt
```

- **OPCION**: Puede tener dos posibles valores. El primero **DNS:HOSTNAME** donde HOSTNAME es el nombre del servidor al que se apunta o **IP:IP_SERVIDOR** donde IP_SERVIDOR es una IP balanceada dentro del cluster

Una vez se ejecuta el comando se debe diligenciar la información solicitada y una vez finalizado quedarán dos archivos en el directorio certs

```bash
[root@master1 registry]# ls -lh certs/
total 8.0K
-rw-r--r-- 1 root root 2.1K Dec 11 08:41 registry.crt
-rw------- 1 root root 3.2K Dec 11 08:40 registry.key
[root@master1 registry]#
```

---

### **Paso 5: Crear un secret**

Como paso siguiente se debe crear un secret el cual contendra el contenido de los dos archivos creados previamente:

```bash
kubectl create secret tls registry-cert --cert=certs/registry.crt --key=certs/registry.key -n registry
```

---

### **Paso 6: Copiar certificado en nodos del cluster**

Para poder permitir el acceso a las imágenes contenidas en el registry, se debe copiar el **registry.crt** a todos los nodos del cluster y reiniciar el servicio **kuecetl.service**.

```bash
ansible -i inventario all -m copy -a "src=certs/registry.crt dest=/etc/pki/ca-trust/source/anchors/" --ask-pass

ansible -i inventario all -m shell -a "update-ca-trust" --ask-pass

ansible -i inventario all -m shell -a "systemctl restart kubelet.service" --ask-pass
```

---

### **Paso 7: Crear el pod registry**

Se crea el deployment del registry el cual usa el PVC y el secret hechos previamente:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: registry
  name: registry
  namespace: registry
spec:
  replicas: 1
  selector:
    matchLabels:
      run: registry
  template:
    metadata:
      labels:
        run: registry
    spec:
      containers:
      - name: registry
        image: registry:2
        ports:
        - containerPort: 5000
        env:
        - name: REGISTRY_HTTP_TLS_CERTIFICATE
          value: "/certs/tls.crt"
        - name: REGISTRY_HTTP_TLS_KEY
          value: "/certs/tls.key"
        volumeMounts:
        - name: registry-certs
          mountPath: "/certs"
          readOnly: true
        - name: registry-data
          mountPath: /var/lib/registry
          subPath: registry
      volumes:
      - name: registry-certs
        secret:
          secretName: registry-cert
      - name: registry-data
        persistentVolumeClaim:
          claimName: nfs-pvc
```

Aplica el manifiesto del Pod:

```bash
kubectl apply -f registry.yml
```

Verifica que el Pod esté funcionando:

```bash
[root@master1 registry]# kubectl get pod -n registry
NAME                        READY   STATUS      RESTARTS   AGE
registry-65f77cd9c7-tnz42   1/1     Running     0          5h44m
[root@master1 registry]# 
```

---

### **Paso 8: Crear Servicio**

Para finalizar se crea el servicio el cual expondrá el servidor registry al cluster por el puerto **31320**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: registry-service
  namespace: registry
spec:
  type: NodePort
  selector:
    run: registry
  ports:
    - port: 5000
      nodePort: 31320
      protocol: TCP
      targetPort: 5000
```

---

### **Verificación**

1. Para realizar verificación se debe instalar docker en una de las máquinas donde se copiaron los certificados y lo primero que haremos es bajar una imagen; en este caso **alpine** 

```bash
[root@endpoint ~]# docker pull alpine
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Resolved "alpine" as an alias (/etc/containers/registries.conf.d/000-shortnames.conf)
Trying to pull docker.io/library/alpine:latest...
Getting image source signatures
Copying blob 38a8310d387e done   | 
Copying config 4048db5d36 done   | 
Writing manifest to image destination
4048db5d36726e313ab8f7ffccf2362a34cba69e4cdd49119713483a68641fce
[root@endpoint ~]#
```

2. Ahora se debe crear un tag de la imagen descargada con la opción que tomamos en el paso 4. En este ejemplo se realiza con el hostname:

```bash
[root@endpoint ~]# docker tag alpine endpoint:31320/alpine:latest
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
[root@endpoint ~]# 
```

Para verificar lo hacemos de la siguiente manera:

```bash
[root@endpoint ~]# docker images
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
REPOSITORY                TAG         IMAGE ID      CREATED     SIZE
endpoint:31320/alpine     latest      4048db5d3672  6 days ago  8.14 MB
docker.io/library/alpine  latest      4048db5d3672  6 days ago  8.14 MB
```

3. Procedemos a subir nuestra imagen al registry creado:

```bash
[root@endpoint ~]# docker push endpoint:31320/alpine
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Getting image source signatures
Copying blob 3e01818d79cd done   | 
Copying config 4048db5d36 done   | 
Writing manifest to image destination
[root@endpoint ~]# 
```

---