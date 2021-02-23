# Game Of Pods

## King's Landing

### TL;DR for King's Landing

Ready Files/commands: 
1 - [create-ns.yaml](./kings-landing/create-ns.yaml)
2 - [voting-dep.yaml](./kings-landing/voting-dep.yaml)
3 - [vote-service.yaml](./kings-landing/vote-service.yaml)
4 - [redis-dep.yaml](./kings-landing/redis-dep.yaml)
5 - `kubectl expose deploy redis-deployment --name redis --port=6379 --target-port=6379 -n vote`
6 - [worker-dep.yaml](./kings-landing/worker-dep.yaml)
7 - [db-dep.yaml](./kings-landing/db-dep.yaml)
8 - `kubectl expose deploy db-deployment --name db --port=5432 --target-port=5432 -n vote`
9 - [result-dep.yaml](./kings-landing/result-dep.yaml)
10 - [result-service.yaml](./kings-landing/result-service.yaml)
  

### Create Namespace
```
kubectl create ns vote
```

### Vote 

Create a `Deploy Object` called `~/voting-dep.yaml`:
```
kubectl create deploy vote-deployment --image=kodekloud/examplevotingapp_vote:before -n vote --dry-run -o yaml > ~/voting-dep.yaml
```
Open `~/voting-dep.yaml` and change the containers name at `spec.template.spec.containers[0].name` to `examplevotingapp-vote` then `kubectl apply -f ~/voting-dep.yaml`

Create a `Service Object` called `vote-service.yaml`:
```
kubectl create svc nodeport vote-service --node-port=31000 --tcp=5000:80 -n vote --dry-run -o yaml > vote-service.yaml
```

Open `vote-service.yaml` and change `spec.selector.app` to `vote-deployment` then `kubectl apply -f vote-service.yaml`

### Redis

Create a `Deploy Object` called `~/redis-dep.yaml`:
```
kubectl create deploy redis-deployment --image=redis:alpine -n vote --dry-run -o yaml > ~/redis-dep.yaml
```

Open `redis-dep.yaml` and add in `spec.template.spec` in the same indent from `.containers`: 
```
...
    volumes:
    - name: redis-data
      emptyDir: {}
...
```
Also, add in `spec.templates.spec.containers[0]`:
```
      resources: {}
      volumeMounts:
      - name: redis-data
        mountPath: "/data"
```
Then `kubectl apply -f ~/redis-dep.yaml`

Create a `Service` using `expose` command:

```
kubectl expose deploy redis-deployment --name redis --port=6379 --target-port=6379 -n vote
```

### Worker

Create a `Deploy Object` called `~/worker-dep.yaml`: 
```
kubectl create deploy worker --image=kodekloud/examplevotingapp_worker -n vote --dry-run -o yaml > ~/worker-dep.yaml
```

Open `~/worker-dep.yaml` and change the containers name at `spec.template.spec.containers[0].name` to `examplevotingapp-worker` then `kubectl apply -f ~/worker-dep.yaml` . 

### Database

Create a `Deploy Object` called `~/db-dep.yaml`: 
```
kubectl create deploy db-deployment --image=postgres:9.4 -n vote --dry-run -o yaml > ~/db-dep.yaml
```

Open `db-dep.yaml` and add in `spec.template.spec` in the same indent from `.containers`: 
```
...
    volumes:
    - name: db-data
      emptyDir: {}
...
```

Also, add in `spec.templates.spec.containers[0]`:
```
      resources: {}
      volumeMounts:
      - name: db-data
        mountPath: "/var/lib/postgresql/data"
```

INFO , the postgres container needs an ENV Variable, add in `.spec.templates.spec.containers[0]`:
```
    name: postgres
    env:
    - name: POSTGRES_HOST_AUTH_METHOD
      value: "trust"
    resources: {}
```

Then `kubectl apply -f ~/db-dep.yaml`

Create a `Service` using `expose` command:
```
kubectl expose deploy db-deployment --name db --port=5432 --target-port=5432 -n vote
```

### Result

Create a `Deploy Object` called `~/result-dep.yaml`: 
```
kubectl create deploy result-deployment --image=kodekloud/examplevotingapp_result:before -n vote --dry-run -o yaml > ~/result-dep.yaml
```
Open `~/result-dep.yaml` and change the containers name at `spec.template.spec.containers[0].name` to `examplevotingapp-result` then `kubectl apply -f ~/result-dep.yaml`.

Create a `Service Object` called `~/result-service.yaml`:
```
kubectl create svc nodeport result-service --node-port=31001 --tcp=5001:80 -n vote --dry-run -o yaml > result-service.yaml
```

Open `~/result-service.yaml` and change `spec.selector.app` to `result-deployment` then `kubectl apply -f ~/result-service.yaml`.


Then click in `Check` and get the `Magic chant` that start with `Hen desarrollo...`.


## Iron Gallery

### TL;DR for Iron Gallery

Ready Files/commands: 
1 - [iron-dep.yaml](./iron-gallery/iron-dep.yaml)
2 - [iron-gallery-svc.yaml](./iron-gallery/iron-gallery-svc.yaml)
3 - [iron-db-dep.yaml](./iron-gallery/iron-db-dep.yaml)
4 - [iron-db-service.yaml](./iron-gallery/iron-db-service.yaml)
5 - [iron-netpolicy.yaml](./iron-gallery/iron-netpolicy.yaml)
6 - [ingress-object.yaml](./iron-gallery/ingress-object.yaml)


### Ingress Iron

Create a `Deploy Object` called `~/iron-dep.yaml`: 
```
kubectl create deploy iron-gallery --image=kodekloud/irongallery:2.0 --dry-run -o yaml > iron-dep.yaml
```

Open the `~/iron-dep.yaml` file and replace the current value in `spec.selector.matchLabels` , `spec.template.metadata.labels` and `.metadata.labels` to:
```
 run: iron-gallery
```

Add in `spec.template.spec`:
```
volumes:
- name: config
  emptyDir: {}
- name: images
  emptyDir: {}
```

Add the in `spec.template.spec.containers[0]`:
```
resources:
 limits:
  memory: "100Mi"
  cpu: "50m"
volumeMounts:
- name: config
  mountPath: /usr/share/nginx/html/data
- name: images
  mountPath: /usr/share/nginx/html/uploads
```

Save and `kubectl apply -f ~/iron-dep.yaml` . 

Create a `Service Object` called `iron-gallery-svc.yaml`: 
```
kubectl create svc nodeport iron-gallery-service --node-port=30099 --tcp=80:80 --dry-run -o yaml > iron-gallery-svc.yaml
```

Change the `spec.selector` field to: 
```
selector:
    run: iron-gallery
```

Save and `kubectl apply -f iron-gallery-svc.yaml` . 


### DB Iron

Create a `Deploy Object` called `iron-db-dep.yaml`: 
```
kubectl create deploy iron-db --image=kodekloud/irondb:2.0 --dry-run -o yaml > iron-db-dep.yaml
```

Open `iron-db-dep.yaml` file and replace the current value in `spec.selector.matchLabels`, `spec.template.metadata.labels` and `metadata.labels` to:
```
db: mariadb
```

Add in `spec.template.spec`:
```
volumes:
- name: db
  emptyDir: {}
```

Add the in `spec.template.spec.containers[0]`:
```
env:
- name: MYSQL_ROOT_PASSWORD
  value: "Braavo"
- name: MYSQL_DATABASE
  value: "lychee"
- name: MYSQL_USER
  value: "lychee"
- name: MYSQL_PASSWORD
  value: "lychee"
resources: {}
volumeMounts:
- name: db
  mountPath: "/var/lib/mysql"
```

Save and `kubectl apply -f iron-db-dep.yaml` . 

Create a `Service Object` called `iron-db-service.yaml`: 
```
kubectl create svc clusterip iron-db-service --tcp=3306:3306 -o yaml --dry-run > iron-db-service.yaml
```

Change the `spec.selector` field to:
```
selector:
 db: mariadb
```

Then `kubectl apply -f iron-db-service.yaml` .

### Netpolicy

https://kubernetes.io/docs/concepts/services-networking/network-policies/#networkpolicy-resource


Open `vim iron-netpolicy.yaml` and add the following resources: 

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: iron-gallery-firewall
  namespace: default
spec:
  podSelector:
    matchLabels:
      db: mariadb
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
       matchLabels:
        run: iron-gallery
    ports:
    - protocol: TCP
      port: 3306
```

Then `kubectl apply -f netpolicy.yaml`.


### Ingress Object

https://kubernetes.io/docs/concepts/services-networking/ingress/#the-ingress-resource

Open `vim ingress-object.yaml` and add the following resources: 

```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: iron-gallery-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: iron-gallery-braavos.com
    http:
      paths:
      - path: /
        backend:
          serviceName: iron-gallery-service
          servicePort: 80
```
Or use the file: 
[ingress-object.yaml]('./iron-gallery/ingress-object.yaml')


Then, `kubectl apply -f ingress-object.yaml`. 

Then click in `Check` and get the `Magic chant` that start with `yamlio muño...`.

# Optional!!

The rest of the challenges are in this [Optional file](./Optional.md).