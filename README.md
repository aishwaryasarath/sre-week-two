# Week 2 - Troubleshooting a production incident, root cause, resolve  & create action items

## Scenario
Your teammate at UpCommerce, has just pushed an update after some changes made to the site's code by the development team. Ever since that push, you have been getting pager alerts that the UpCommerce.com service is down. As SRE team lead, it is your job to handle and resolve problems that happen in the UpCommerce Kubernetes cluster. You will also need to keep your Incident Commander (IC) and Communications Lead (CL) up to date so they can manage UpCommerce's users

Deploy the code in this repo following the same steps as outlined in [this](https://github.com/aishwaryasarath/sre-task-repo) repo.

## Root cause
```
upcommerce-app-two-65745f9d8b-j9scl -n sre
Name:             upcommerce-app-two-65745f9d8b-j9scl
Namespace:        sre
Priority:         0
Service Account:  default
Node:             <none>
Labels:           app=upcommerce-app-two
                  pod-template-hash=65745f9d8b
Annotations:      <none>
Status:           Pending
IP:               
IPs:              <none>
Controlled By:    ReplicaSet/upcommerce-app-two-65745f9d8b
Containers:
  upcommerce:
    Image:      uonyeka/upcommerce:v3
    Port:       5000/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     2
      memory:  4Gi
    Requests:
      cpu:        2
      memory:     4Gi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rzgwh (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  kube-api-access-rzgwh:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Guaranteed
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  24s   default-scheduler  0/1 nodes are available: 1 Insufficient cpu. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..
```

## Cause
Codespace free tier can have only a max of 1 cpu. Anything more than that fails.


## Fix

### Current code of deployment
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: upcommerce-app-two
spec:
  replicas: 1
  selector:
    matchLabels:
      app: upcommerce-app-two
  template:
    metadata:
      labels:
        app: upcommerce-app-two
    spec:
      containers:
        - name: upcommerce
          image: uonyeka/upcommerce:v3
          ports:
            - containerPort: 5000
          imagePullPolicy: Always
          resources:
            limits:
              cpu: "10"
              memory: "4Gi"

```

### Change the cpu as below
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: upcommerce-app-two
spec:
  replicas: 1
  selector:
    matchLabels:
      app: upcommerce-app-two
  template:
    metadata:
      labels:
        app: upcommerce-app-two
    spec:
      containers:
        - name: upcommerce
          image: uonyeka/upcommerce:v3
          ports:
            - containerPort: 5000
          imagePullPolicy: Always
          resources:
            limits:
              cpu: "1"
              memory: "4Gi"

```
