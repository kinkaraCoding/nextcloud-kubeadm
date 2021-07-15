

## Create namespace nextcloud

Change your namespace from using `default` to `nextcloud`.
For all resources for which no explicit namespace was set, are now created in namespace `nextcloud` 

``` shell
kubectl create namespace nextcloud
```

``` shell
kubectl config set-context --current --namespace=nextcloud
```


## Create deployment secrets


``` shell
kubectl create secret generic nextcloud-secrets \
    --from-literal=MYSQL_ROOT_PASSWORD=CHANGE_ME \
    --from-literal=MYSQL_USER=CHANGE_ME \
    --from-literal=MYSQL_PASSWORD=CHANGE_ME \
    --from-literal=NEXTCLOUD_ADMIN_USER=CHANGE_ME \
    --from-literal=NEXTCLOUD_ADMIN_PASSWORD=CHANGE_ME
```
## Create deployment configMap

``` yaml
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nextcloud-configmap
data:
  MYSQL_DATABASE: CHANGE_ME
  MYSQL_HOST: CHANGE_ME
  NEXTCLOUD_TRUSTED_DOMAINS: CHANGE_ME
  ADMINER_DESIGN: pepa-linha
  ADMINER_DEFAULT_SERVER: CHANGE_ME (same as MYSQL_HOST)
EOF
```

## Create a deployment PV
For PersistenVolume we use `type: local` to store our user data on a worker node local filesystems.
Make sure the path `/media/k8s-data` extis on your worker node!

``` yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nextcloud-local-pv
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 15Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: "/media/k8s-data"
EOF
```

``` shell
kubectl get pv nextcloud-local-pv
``` 

``` shell
NAME                 CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM   STORAGECLASS   REASON   AGE
nextcloud-local-pv   15Gi       RWO            Retain           Bound             manual                  2m
``` 

## Create deployment PVC
With this PersistentVolumesClaim(PVC) we request a storage of 10Gi from our PersistenVolume(PV)
provided by microk8s´s Storage addon.

``` shell
kubectl apply -f https://raw.githubusercontent.com/kinkaraCoding/nextcloud-kubeadm/master/nextcloud-pvc.yaml
```

Check if your PVC is bound properly

``` shell
kubectl get pvc nextcloud-shared-storage-claim
``` 

``` shell
NAME                             STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS        AGE
nextcloud-shared-storage-claim   Bound    pvc-xxx   10Gi       RWO            microk8s-hostpath   9s
``` 


## Create deployment nextcloud-db
Deploy a mariadb database as backend for nextcloud-server

``` shell
kubectl apply -f  https://raw.githubusercontent.com/kinkaraCoding/nextcloud-kubeadm/master/nextcloud-db.yaml
```

## Create deployment nextcloud-server
Deploy nextcloud frontend server


``` shell
kubectl apply -f https://raw.githubusercontent.com/kinkaraCoding/nextcloud-kubeadm/master/nextcloud-server.yaml
```  


## Create deployment nextcloud-adminer
Deploy PHP based web ui for nextcloud-db
``` shell
kubectl apply -f https://raw.githubusercontent.com/kinkaraCoding/nextcloud-kubeadm/master/nextcloud-adminer.yaml
```  


## Create Ingress
Put your correct domain in both host sections for `host: nextcloud.example.com`and `host: adminer.example.com`

``` shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud-ingress
spec:
  rules:
  - host: nextcloud.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nextcloud-server
          servicePort: 80
  - host: adminer.example.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nextcloud-server
          servicePort: 8080
EOF
```  

## Create a Secret of your certificate
If you dont have a trusted certificate you can use [mkcert](https://github.com/FiloSottile/mkcert) to
create a self signed one. Certificate and private key have to encode in base64 to create a K8s secret
object.


``` shell
base64 *.crt
```  

``` shell
base64 *.key
```  


``` shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: ingress-cert
data:
  tls.crt: |
    BASE64_ENCODED_STRING_OF_YOUR_CERTIFICATE
  tls.key: |
    BASE64_ENCODED_STRING_OF_YOUR_PRIVATE_KEY
EOF
```


## Create Ingress with TLS 
Put your correct domain in both host sections for `host: nextcloud.example.com`and `host: adminer.example.com`

``` shell
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nextcloud-ingress-prod-tls
  annotations:
    kubernetes.io/ingress.class: "nginx"    
spec:
  tls:
  - hosts:
    - nextcloud-k8s.example.com
    - adminer-k8s.example.com
    secretName: ingress-cert
  rules:
  - host: nextcloud-k8s.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: nextcloud-server
            port:
              number: 80
  - host: adminer-k8s.example.com
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: nextcloud-adminer
            port:
              number: 8080
EOF
```
