# Optional

## Redis Islands

https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#components

Open `vim redis-sts.yaml` and add the following: 

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  selector:
    matchLabels:
      app: redis
  serviceName: "redis-cluster-service"
  replicas: 6 
  template:
    metadata:
      labels:
        app: redis
    spec:
      terminationGracePeriodSeconds: 10
      volumes: 
      - configMap:
          defaultMode: 0755
          name: redis-cluster-configmap
        name: conf
      containers:
      - name: redis
        image: redis:5.0.1-alpine
        command:
        - /conf/update-node.sh
        - redis-server
        - /conf/redis.conf
        env: 
        - name: POD_IP
          valueFrom: 
           fieldRef:
             fieldPath: status.podIP
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
```

Open `vim pvs.yaml` and add this: 

```
apiVersion: v1
kind: PersistentVolume
metadata:
 name: redis01
spec:
 accessModes: ["ReadWriteOnce"]
 capacity:
   storage: "1Gi"
 hostPath:
  path: /redis01
---
apiVersion: v1
kind: PersistentVolume
metadata:
 name: redis02
spec:
 accessModes: ["ReadWriteOnce"]
 capacity:
   storage: "1Gi"
 hostPath:
  path: /redis02
---
apiVersion: v1
kind: PersistentVolume
metadata:
 name: redis03
spec:
 accessModes: ["ReadWriteOnce"]
 capacity:
   storage: "1Gi"
 hostPath:
  path: /redis03
---
apiVersion: v1
kind: PersistentVolume
metadata:
 name: redis04
spec:
 accessModes: ["ReadWriteOnce"]
 capacity:
   storage: "1Gi"
 hostPath:
  path: /redis04
---
apiVersion: v1
kind: PersistentVolume
metadata:
 name: redis05
spec:
 accessModes: ["ReadWriteOnce"]
 capacity:
   storage: "1Gi"
 hostPath:
  path: /redis05
---
apiVersion: v1
kind: PersistentVolume
metadata:
 name: redis06
spec:
 accessModes: ["ReadWriteOnce"]
 capacity:
   storage: "1Gi"
 hostPath:
  path: /redis06
```

With the files `pvs.yaml` and `redis-sts.yaml` created but not applied, run the following command: 
```
$ ssh $(kubectl get nodes -o wide | grep node | awk '{print $6}') 'mkdir -p /redis0{1..6}'
```

Now apply the config files ` kubectl apply -f pvs.yaml` and `kubectl apply -f redis-sts.yaml`.

After applied, execute: 
```
kubectl exec -it redis-cluster-0 -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis -o jsonpath='{range.items[*]}{.status.podIP}:6379 ')
```
When prompted, write `yes` then `Enter`. 

Create a `Service Object` in a file called `redis-svc.yaml`: 
```
kubectl create svc clusterip redis-cluster-service --tcp 6379:6379,16379:16379 --dry-run -o yaml > redis-svc.yaml
```

Change the names of the ports in `spec.ports`: 
```
  - name: client
    port: 6379
    protocol: TCP
    targetPort: 6379
  - name: gossip
    port: 16379
    protocol: TCP
    targetPort: 16379
```
Then `kubectl apply -f redis-svc.yaml`. 

## Braavo

Create Folders in node

```
ssh $(kubectl get nodes -o wide | grep node | awk '{print $6}') 'mkdir /drupal-data'

ssh $(kubectl get nodes -o wide | grep node | awk '{print $6}') 'mkdir /drupal-mysql-data'
```
Create a file called `pvs.yaml` and add: 
```
apiVersion: v1
kind: PersistentVolume
metadata:
 name: drupal-pv
spec:
 accessModes: ["ReadWriteOnce"]
 capacity:
   storage: "5Gi"
 hostPath:
  path: /drupal-data
---
apiVersion: v1
kind: PersistentVolume
metadata:
 name: drupal-mysql-pv
spec:
 accessModes: ["ReadWriteOnce"]
 capacity:
   storage: "5Gi"
 hostPath:
  path: /drupal-mysql-data
```

Create a file called `pvcs.yaml` and add: 
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: drupal-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
     storage: "5Gi"
  volumeName: drupal-pv
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: drupal-mysql-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
     storage: "5Gi"
  volumeName: drupal-mysql-pv
```

Create a secret: 
```
kubectl create secret generic drupal-mysql-secret --from-literal=MYSQL_ROOT_PASSWORD=root_password --from-literal=MYSQL_DATABASE=drupal-database --from-literal=MYSQL_USER=root
```

Create a file called `drupal-db.yaml` and add:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: drupal-mysql
  name: drupal-mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: drupal-mysql
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: drupal-mysql
    spec:
      containers:
      - image: mysql:5.7
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
           secretKeyRef:
            name: drupal-mysql-secret
            key: MYSQL_ROOT_PASSWORD
        - name: MYSQL_DATABASE
          valueFrom:
           secretKeyRef:
            name: drupal-mysql-secret
            key: MYSQL_DATABASE
        - name: MYSQL_USER
          valueFrom:
           secretKeyRef:
            name: drupal-mysql-secret
            key: MYSQL_USER
        resources: {}
        volumeMounts:
        - name: drupal-mysql-pvc
          mountPath: /var/lib/mysql
          subPath: dbdata
      volumes:
      - name: drupal-mysql-pvc
        persistentVolumeClaim:
         claimName: drupal-mysql-pvc
```

Apply the `kubectl apply -f drupal-db.yaml` and now `expose` the deployment using:
```
kubectl expose deploy drupal-mysql --name drupal-mysql-service --port 3306 --target-port 3306
```

Create a file called `drupal-dep.yaml` and add:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: drupal
  name: drupal
spec:
  replicas: 1
  selector:
    matchLabels:
      app: drupal
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: drupal
    spec:
      containers:
      - image: drupal:8.6
        name: drupal
        resources: {}
        volumeMounts:
        - name: drupal-pvc
          mountPath: /var/www/html/modules
          subPath: modules
        - name: drupal-pvc
          mountPath: /var/www/html/sites
          subPath: sites
        - name: drupal-pvc
          mountPath: /var/www/html/profiles
          subPath: profiles
        - name: drupal-pvc
          mountPath: /var/www/html/themes
          subPath: themes
      initContainers:
      - name: init-sites-volume
        image: drupal:8.6
        command:
        - /bin/bash
        - -c
        args:
        - 'cp -r /var/www/html/sites/ /data/; chown www-data:www-data /data/ -R'
        resources: {}
        volumeMounts:
        - name: drupal-pvc
          mountPath: /data
      volumes:
      - name: drupal-pvc
        persistentVolumeClaim:
         claimName: drupal-pvc
status: {}
```
Apply using `kubectl apply -f drupal-dep.yaml` and create a file called `drupal-svc.yaml` using:
```
kubectl create svc nodeport drupal-service --node-port=30095 --tcp=80:80 --dry-run -o yaml > drupal-svc.yaml
```
Open the file `drupal-svc.yaml` and `spec.selector.app` to `drupal` and `kubectl apply -f drupal-svc.yaml`  

## Pento

In this test kube-api-server is not ready, first change the port in `/root/.kube/config` from `2379` to `6443`. 
Now check the file `/etc/kubernetes/manifests/kube-apiserver.yaml`  the Certificate Authority file is wrong , fix setting the property as `--client-ca-file=/etc/kubernetes/pki/ca.crt`. Restart `Kubelet` service, doing: 
```
systemctl restart kubelet
```

Now lets deploy the application, create a file `vim pv.yaml` and add:

```
apiVersion: v1
kind: PersistentVolume
metadata:
 name: data-pv
spec:
 accessModes: ["ReadWriteMany"]
 capacity:
   storage: "1Gi"
 hostPath:
  path: /web
```

Create a file `vim pvc.yaml` and add:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
     storage: "1Gi"
  volumeName: data-pv
```

Create a file `gop-pod.yaml`, using:
```
kubectl run --generator=run-pod/v1 gop-fileserver --image=kodekloud/fileserver --dry-run -o yaml > gop-pod.yaml
```
And add:

```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: gop-fileserver
  name: gop-fileserver
spec:
  containers:
  - image: kodekloud/fileserver
    name: gop-fileserver
    resources: {}
    volumeMounts:
    - name: data-store
      mountPath: /web
  volumes:
  - name: data-store
    persistentVolumeClaim:
      claimName: data-pvc
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Create a file called `gop-svc.yaml`: 
```
kubectl create svc nodeport gop-fs-service --node-port=31200 --tcp=8080:8080 --dry-run -o yaml > gop-svc.yaml
```
Then add:
```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: gop-fs-service
  name: gop-fs-service
spec:
  ports:
  - name: 8080-8080
    nodePort: 31200
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    run: gop-fileserver
  type: NodePort
```

## Tyro

Type those commands to create a user in the file `/root/.kube/config`: 
```
kubectl config set-credentials drogo --client-key=/root/drogo.key --client-certificate=/root/drogo.crt
kubectl config set-context developer --user=drogo --cluster=kubernetes
```
Now create some permissions to these user: 
```
kubectl create role developer-role --verb=* --resource=services,pods,persistentvolumeclaims -n development
kubectl create rolebinding developer-rolebinding --user=drogo --role=developer-role -n development
```

Create a PVC file `vim pvc.yaml`, using the following data:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jekyll-site
  namespace: development
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
     storage: "1Gi"
  volumeName: jekyll-site
```
Then apply using `kubectl apply -f pvc.yaml`. 

Create a file using `jekyll-pod.yaml`, using:
```
kubectl run --generator=run-pod/v1 jekyll --image=kodekloud/jekyll-serve --dry-run -o yaml > jekyll-pod.yaml
```
Add: 
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: jekyll
  name: jekyll
spec:
  containers:
  - image: kodekloud/jekyll-serve
    name: jekyll
    resources: {}
    volumeMounts:
    - name: site
      mountPath: /site
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  initContainers:
  - name: copy-jekyll-site
    image: kodekloud/jekyll
    command:
    - jekyll
    - new
    - /site
    resources: {}
    volumeMounts:
    - name: site
      mountPath: /site
  volumes:
  - name: site 
    persistentVolumeClaim:
     claimName: jekyll-site
status: {}
```
Then apply `kubectl apply -f jekyll-pod.yaml`. 

Create a SVC file called `jekyll-svc.yaml` using: 
```
kubectl create svc nodeport jekyll --node-port=30097 --tcp=8080:4000 -n development --dry-run -o yaml > jekyll-svc.yaml
```
Add: 
```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: jekyll
  name: jekyll
  namespace: development
spec:
  ports:
  - name: 8080-4000
    nodePort: 30097
    port: 8080
    protocol: TCP
    targetPort: 4000
  selector:
    run: jekyll
  type: NodePort
```
Then apply `kubectl apply -f jekyll-svc.yaml`.

For last do: 
```
kubectl config use-context developer
```