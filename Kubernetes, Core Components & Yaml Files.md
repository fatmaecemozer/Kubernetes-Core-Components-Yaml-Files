## Kubernetes, Core Components & Yaml Files by FEO

Open source orchestration tool developed by Google.
Manages containerized applications in different deployment environments. 
- High Availability or no down time
- Scalability or high perfomance 
- Disaster Recovery or backups and restore

*Pod:* smallest part of k8s abstraction over container.

* Usually one application per Pod. 
*  Each pod has its own IP address. New IP address on  re-creation. 

*Service:* static (permanent) IP address that can be attached to each pod. Even if the pod dies the service and its IP address survives. 
Everything should be replicated. The replica is connected to the same Service, so it also works as a Load Balancer catches the request and forwards it to the least busy pod. 

*Deployment:* blueprint for application pods which also helps other configurations.
DB can't be replicated via Deployment. 

*StatefulSet:* for stateFUL applications, makes sure all DB replicas reads and writes are synchronized to avoid data inconsistency. 

***Instead of deploying StatefulSet, DB are often hosted outside of K8s cluster!***

![](https://i.imgur.com/qXErxxC.png)


*Ingress:* To be able to talk with the application from a browser with a secure protocol and a domain name, Ingress component used. Instead of Service, request goes first to Ingress and it does the forwarding to the Service.

*Config Map:* external configuration data for the application. 

*Secret:* used to store credentials, base64 encoded format. 

*Volumes:* attaches physical storage either on the local machine or on the remote storage. K8s doesn't manage data persistance. 


![](https://i.imgur.com/IyZO8zo.png)

#### K8s Cluster Processes:
Each worker node has multiple pods on it. 
Worker node components;
- Container Runtime, docker etc.
- Kubelet, interacts with both the container and node, runs or starts a pod and assign resources
- KubeProxy, works as network proxy 

Master node components;
- Api-Server, cluster gateway, authentication, authorization
- etcd, holds cluster current state, key value storage
- Schedular, for assigning pods to nodes
- Controller-Manager, cluster state tracking with the help of api-server, 
Desired State <-> Current State comparison 


![](https://i.imgur.com/SrzV0Ec.png)

Deployment -> abstraction over Pods;
`kubectl create deployment NAME --image=image` 

Use configuration file for CRUD;
`kubectl apply -f [file name] #apply a configuration file `
`kubectl delete -f [file name] #delete with configuration file` 

Debugging pods; 
`kubectl log [pod name] #log to console`
`kubectl exec -it [pod name] -- bin/bash #get interactive terminal`
`kubectl describe pod [pod name] #get info about pod`

Each configuration file has 3 main parts;
- metadata: meta info 
- specification: attributes are specific for the kind
- status: automatically generated and added by kubernetes, comes from etcd

YAML *'Yet Another Markup Language'* Configuration Files;
- strict indentation syntax!
- store the config file with the code or own git repository

nginx deployment.yaml file example;
```
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
  labels: 
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1 #tells deployment to run 1 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 8080
```
nginx service.yaml file example;
```
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
    targetPort: 8080
```

#### Mongo Example;
![](https://i.imgur.com/ugxz1v9.png)

#### MongoDB deployment;
mongo.yaml
```
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: mongodb-deployment
  labels: 
    app: mongodb
spec:
  selector:
    matchLabels:
      app: mongodb
  replicas: 2 #tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom: #username and password should be stored in Secret 
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom: 
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password 
---
apiVersion: apps/v1 
kind: Service
metadata:
  name: mongodb-service
spec:
  selector: #to connect to pod through label
    app: mongodb 
  ports:
    - protocol: TCP
      port: 27017 #service port 
      targetPort: 27017 #container port of deployment can be different from service port 
```

**Secret lives in K8s, not in the repository!**
mongo-secret.yaml
```
apiVersion: v1 
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque #default
data: #actual content for key-value pairs must be base64 encoded
    mongo-root-username: vvvv
    mongo-root-password: vvvv
```

base64 encoded username and password can be obtained from terminal;
`echo -n 'username' | base64`
`echo -n 'password' | base64`

Secret file must be applied first;
`kubectl apply -f mongo-secret.yaml`
`kubectl get secret`

Deployment & Service file must be applied;
`kubectl apply -f mongo.yaml`
`kubectl get deployment`
`kubectl get service`
`kubectl describe service [service name]` 
`kubectl get pod -o wide`
`kubectl get all | grep mongodb`

#### Mongo Express;
mongo-express.yaml
```
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: mongo-express
  labels: 
    app: mongo-express
spec:
  selector:
    matchLabels:
      app: mongo-express
  replicas: 2 #tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom: #username and password should be stored in Secret 
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom: 
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password 
        - name: ME_CONFIG_MONGODB_SERVER #will be using config map 
          valueFrom: 
            configMapKeyRef:
              name: mongodb-configmap
              key: database_url
---
apiVersion: apps/v1 
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector: 
    app: mongo-express
  type: LoadBalancer #except external requests 
  #LoadBalancer assigns in addition an External IP
  ports:
    - protocol: TCP
      port: 8081 
      targetPort: 8081
      nodePort: 30000 #between 30000-32767
```

mongo-configmap.yaml
```
apiVersion: v1 
kind: ConfigMap
metadata:
  name: mongodb-configmap
data: 
  database_url: mongodb-service
```
ConfigMap file must be applied first;
`kubectl apply -f mongo-configmap.yaml`
`kubectl get configmap`

Deployment & Service file must be applied;
`kubectl apply -f mongo-express.yaml`
`kubectl get deployment`
`kubectl get service`

External IP will be accesible through external requests.

![](https://i.imgur.com/zgvfvng.png)

#### Namespaces: 
virtual cluster inside a cluster
- group resources
- avoid conflicts
- re-use components, resource sharing
- limit resources and access

*kube-system* #do not create or modify in here!
*kube-public* #publicly accessible data
*kube-node-lease* #holds info of nodes, heartbeats
*default*
`kubectl create namespace NAME`

```
apiVersion: v1 
kind: ConfigMap
metadata:
  name: mongodb-configmap
  namespace: newnsname #defining a namespace in config file
data: 
  database_url: mongodb-service
```

Can change the active namespace with kubens!

#### Kubernetes Ingress
IP address and port number is not open to see from outside world with use of ingress. 

*http://my-app.com*
```
apiVersion: networking.k8s.io/v1beta1 
kind: Ingress
metadata:
  name: myapp-ingress
spec:
  rules:
  - host: myapp.com #user will enter in browser, valid domain address
    http:
      paths: #url path
      - backend:
          serviceName: myapp-internal-service
          servicePort: 8080
```
- Map domain name to Node's IP Adress, which is the entrypoint !

Ingres and Internal Service Configuration:
```
apiVersion: v1 
kind: Service
metadata:
  name: myapp-internal-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```
- No nodePort needed in internal service!

***K8s Nginx Ingress Controller:*** an implementation that evaluates and processes Ingress rules.
Entrypoint to cluster that manages redirections.

External request that coming from the browser first will hit the LoadBalancer, and it will redirect it to Ingress Controller Pod as the best practise implementation !

Install Ingress Controller in Minikube:
`minikube addons enable ingress`

In Minikube cluster kubernetes-dashboard already exists out-of-the-box. 
`kubectl get namespaces`
*kubernetes-dashboard* #not accessible externally at the moment

`kubectl get all -n kubernetes-dashboard`

dashboard-ingress.yaml
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
spec:
  rules:
  - host: dashboard.com 
    http:
      paths: 
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 80
```
`kubectl apply -f dashboard-ingress.yaml`
`kubectl get ingress -n kubernetes-dashboard`
```
vi /etc/hosts
[IP Address] dashboard.com
```
Will be exposed to outside world.

Multiple paths for the same host:
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-example
spec:
  rules:
  - host: my-app.com 
    http:
      paths: 
      - path: /analytics 
        backend:
          serviceName: analytics-service
          servicePort: 3000
      - path: /shopping
        backend: 
          serviceName: shopping-service
          servicePort: 8080
```
Multiple hosts represents a sub-domains:
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-example
spec:
  rules:
  - host: analytics.myapp.com 
    http:
      paths:  
        backend:
          serviceName: analytics-service
          servicePort: 3000
  - host: shopping.myapp.com 
    http:
      paths: 
        backend:
          serviceName: shopping-service
          servicePort: 8080    
```
Configuring TLS Certificate:
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: tls-example-ingress
spec:
  tls: 
  - host: 
    - myapp.com 
    secretName: myapp-secret-tls #reference of a secret that created in the cluster that holds tls certificate
  rules:
  - host: myapp.com 
    http:
      paths: 
      - path: /
        backend:
          serviceName: myapp-internal-service
          servicePort: 3000   
```   
```
apiVersion: v1 
kind: Secret
metadata:
  name: myapp-secret-tls #!
data: 
    tls.crt: base64 encoded cert
    tls.key: base64 encoded key
type: kubernetes.io/tls
```
1. Data keys need to be named as "tls.crt" and "tls.key"
2. Values are file contents not paths/locations
3. Secret component must be in the same namespace as the Ingress component 

#### Helm main concepts:
- Package manager for Kubernetes, packages yaml files and distributes them in public and private repositories


Helm Charts, a collection of files that describe a related set of Kubernetes resources.
Templating Engine, define a common blueprint to be used as template.

Helm Chart Directory Structure;
```
mychart/ #name of chart
  Chart.yaml #meta info about chart
  values.yaml #values for template files, default values can be overriden
  charts/ #chart dependencies
  templates/ #where actual template files stored
  ...
```
![](https://i.imgur.com/RomWUev.png)
`helm install [chartname]`
`helm upgrade [chartname]`
`helm rollback [chartname]`

Tiller has too much power inside of K8s cluster (create, delete, update) which causes security issues, so in Helm 3 Tiller got removed!

#### Kubernetes Volumes:
Storage that doesn't depend on pod life cycle and available on all nodes and needs to survive even if cluster crashes.
- Persistent Volume: a cluster resource used to store data, created via yaml file, needs actual physical storage on local or remote, cloud environment.

Use that storage in the spec section, specify capacity and parameters. 
Persistent Volumes are accessible to the whole cluster. 
Storage resource is provisioned by Kubernetes Admin.
- Persistent Volume Claim: created via yaml file, PVC claims a volume with specific size.

Pod requests the volume through the PV Claim -> Claim tries to find a volume in cluster -> Volume has the actual storage backend. 
Claims must be in the same namespace as the Pod using it! Volume is mounted into the Pod and when Pod dies, new Pod will be available to access the same storage. 
Claim is provisioned by Kubernetes User. 

Config Map and Secret are local volumes and not created via PV and PVC. You can mount these into Pod as needed.

Different volume types can be used in a Pod by specifying different mount paths under the VolumeMounts. 

- Storage Class: creates Persistent Volumes dynamically when PVC claims it. Also created via yaml file, defined in the storage class component via 'provisioner' attribute, each storage backend has own its provisioner.
Internal Provisioner - 'kubernetes.io'

1. Pod claims storage via PVC 
2. PVC requests storage from SC 
3. SC creates PV that meets the needs of the PVClaim

#### StatefulSet: 
used for stateful applications (apps that stores data)
* stateless applications deployed using Deployment 
* stateful applications deployed using StatefulSet 

![](https://i.imgur.com/ZLyc2Ox.png)

Replica pods are not identical, each have their own identity, stateful set identifies a sticky identity for each pod, so even if the pod dies new one keeps the identity.

To avoid data inconsistency only one pod is allowed to change for both reading and writing which is called Master. 
Replicas are not using the same physical storage, but they have to be synchorinized and have the same data. 
The Worker must know about each change to be up-to-date!
When Master change its data Workers reach those changes.

With persistent storage, data will survive even if all pods die. PV lifecycle is not dependent to other cycles so for the StatefulSet a PV should be created. 

2 characteristics;
1. predictable pod name #mysql-0
2. fixed individual DNS name #mysql-0.svc2

When Pod restarts;
1. IP Address changes
2. name and endpoint stays the same 


#### Kubernetes Services:
Each pod has its own IP Address, everytime Pod gets recreated IP Address of it changes. 
Service provides stable IP Address and LoadBalancing.

ClusterIP Services: the default type, cluster internal ip addresses, service is accessible within the cluster.
`kubectl get pod -o wide #command to check pods ip address`
Once the request comes, it is directed to the service endpoints, pods are identified via selectors to be forwarded to. Port will be chosen via targetPort attribute. 

`kubectl get endpoints`
K8s creates Endpoint object, same name as Service, keeps track of, which Pods are the members/endpoints of the Service. 

Headless Service: client wants to communicate with 1 specific Pod directly, not randomly selected, useful for stateful applications. Client needs to figure out IP addresses of each Pod, DNS Lookup ! returns single IP address as a solution here. 
`ClusterIP: None` should be defined in the yaml file. 

NodePort Service: creates a service that is accessible on a static port on each worker node in the cluster. External traffic has access to a fixed port on each worker node. Not secure for production!

LoadBalancer Service: service becomes accessible externally through cloud providers LoadBalancer. Entrypoint becomes LB first and then traffic flows through NodePort so it will be more secure. 
