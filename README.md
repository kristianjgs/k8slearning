### configurar el bash completion para kubectl

```bash
kubectl completion bash >/etc/bash_completion.d/kubectl
```

### ejecutar pods en todos los nodos master

```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

### Veriicar el estado del cluster

```bash
kubectl get componentstatuses
```

### ejecutar pods en un nodo especifico master

```bash
kubectl taint nodes <node-name> node-role.kubernetes.io/master-
```

### verificar expiración de los certificados

```bash
kubeadm certs check-expiration
```

### Renovar certificados

- Se renuevan los certificados en cada uno de los nodos control plane (masters)
  
  ```bash
  kubeadm certs renew all
  ```

- Se actualiza el certificado de administrador
  
  ```bash
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

### definir el namespace por defecto

- Se identifica el ns por defecto actual

```bash
kubectl config view | grep current-context
```

- Se realiza el cambio al ns deseado

```bash
kubectl config set-context --current --namespace="namespace"
```

### información de un nodo

```bash
kubectl describe node <nombrenodo>
```

### Drain mode

```bash
kubectl drain <nombrenodo> --ignore-daemonsets
```

### uncordon the node (disable drain mode)

```bash
kubectl uncordon <node-to-uncordon>
```

### PODS

- información de un pod

```bash
kubectl describe pod <nombrepod> -n <nombrenamespace>
```

- Ejecutar un comando en un pod

```bash
kubectl exec -n <namespace> <podname> -- <comando>
```

- ingresar a un pod en ejecución

```bash
kubectl exec -it <nombrepod> -- bash
```

- cuando el pod tiene más de un contenedor se hace de la siguiente manera:

```bash
kubectl exec -it <nombrepod> -c <nombreContenedor> -- /bin/bash
```

- ver los logs de un mod

```bash
kubectl logs -n <namespace> <podname>
```

- Copiar archivos desde un pod hacia una máquina local

```bash
kubectl cp -n <namespace> <podname>:/ruta/archivo.txt ./archivo.txt
```

- Copiar archivos desde una máquina local hacia un pod

```bash
kubectl cp /ruta/archivo.txt -n <namespace> <podname>:/ruta/archivo.txt
```

### DEPLOYMENTS

- informacion de un deployment

```bash
kubectl describe <nombredeployment>
```

- ver el status de un cambio

```bash
kubectl rollout status deployment/nginx-deployment
```

- chequear historia de un deployment

```bash
kubectl rollout history deployment/<nombredeployment>
```

- escribir en la causa del cambio para fines informativos

```bash
kubectl annotate deployment/nginx-deployment kubernetes.io/change-cause="image updated to 1.16.1"
```

- chequear una revisión en particular

```bash
kubectl rollout history deployment/nginx-deployment
```

- hacer un rollback a la revisión previa

```bash
kubectl rollout undo deployment/nginx-deployment
```

- hacer un rollback a una revisión específica

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

- pausando y resumiendo un rollout de un deployment

```bash
kubectl rollout pause deployment/nginx-deployment
```

```bash
kubectl rollout resume deployment/nginx-deployment
```

- escalar un deployment aumentando las replicas

```bash
kubectl scale deployment/nginx-deployment --replicas=10
```

- actualizar recursos

```bash
kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
```

- actualizar la imagen

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```

### instalar el dashboard

- Add kubernetes-dashboard repository

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
```

- Deploy a Helm Release named "kubernetes-dashboard" using the kubernetes-dashboard chart

```bash
helm upgrade --install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

- exponer el dashboard para tener acceso desde el navegador

```bash
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```

- hacer bridge con el dashboard

```bash
sudo ssh -L 8443:localhost:8443 vagrant@192.168.121.111
```

- acceder vía web

https://localhost:8443/#/login

- Se debe crear el service account y para esto se debe crear el siguiente YAML:

```bash
vim k8s-dashboard-account.yaml
```

- Se debe copiar el siguiente contenido en el yaml:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```

- Se crea el service account

```bash
kubectl create -f k8s-dashboard-account.yamlc
```

- Ahora se genera el token para poder ingresar al dashboard:

```bash
kubectl -n kube-system create token admin-user
```

El token generado es similar a lo siguiente:

```text
eyJhbGciOiJSUzI1NiIsImtpZCI6Ikt3Tlp4MExodjVyQ3JuWk15aGhGOVFrcEQzcXBNYVZDWlN0Vm55aS1SVnMifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzI5MTAwMTM0LCJpYXQiOjE3MjkwOTY1MzQsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJhZG1pbi11c2VyIiwidWlkIjoiOWUzMTFjZWYtYTI2NC00ZWYxLWIyNWItNGI4YjZiNDY4OTk2In19LCJuYmYiOjE3MjkwOTY1MzQsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTphZG1pbi11c2VyIn0.diXK3pvzTqRp-jFtax3M9lpOZLitkwIPVTOubZxqIfKH12Ihw7CmfQzJWvFmDcQ_TWoNTJF9qXypNKL4JNl-mxx5E5BDoRqJHej4zfkVQRv8B8Co8vkya_E8g3YZLjSg3MSmOvEPB3G7FEhRh4DHXoUiNRuNx9YvkXHaJIPGYC3llnVuWU4IO8EtBBGjh_lrH0W9huK9jsg0Wx09UMvlFVonJwLMrNftX2iBtqF60oslLetpryqFJ9Fiycu9pdgukTp4qgq3qx6_cXIIbcM_NoOrvgT9G9ajLY-wWVZ_YvixXhIJ5-iPvvF4jxWDNHQlRmA0dVe6j83M7o_Xy7s3-w
```

El token expira después de una (1) hora. Si se desea modificar ese tiempo se debe usar el parámetro  **--duration=tiempo**. donde **tiempo** puede ser en horas o minutos. Ejemplo:

```bash
kubectl -n kube-system create token admin-user --duration=876000h
```

- pendientes:
  
  - estudiar HorizontalPodAutoscaler https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
  - prometheus https://prometheus.io/docs/prometheus/latest/installation/
  - instalar herramientas de monitoreo (prometheus, grafana)
  - validar coreos y Tectonic dashboard
  - hacer ejemplos de storage
  - usar helm
  - instalar con crio https://thelinuxnotes.com/index.php/deploying-a-kubernetes-cluster-on-fedora-coreos-with-cri-o/#:~:text=In%20this%20post,%20we%E2%80%99ll%20walk%20through%20setting%20up