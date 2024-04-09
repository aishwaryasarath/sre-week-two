# Week 2 - Troubleshooting a production incident, root cause, resolve  & create action items

## Scenario
Your teammate at UpCommerce, has just pushed an update after some changes made to the site's code by the development team. Ever since that push, you have been getting pager alerts that the UpCommerce.com service is down. As SRE team lead, it is your job to handle and resolve problems that happen in the UpCommerce Kubernetes cluster. You will also need to keep your Incident Commander (IC) and Communications Lead (CL) up to date so they can manage UpCommerce's users

Deploy the code in this repo following the same steps as outlined in [this](https://github.com/aishwaryasarath/sre-task-repo) repo.

## Troubleshooting

### List the deployment, it shows upcommerce-app-two deployment Ready 0/1 whereas it should show Ready 1/1
```
kubectl get deployment -n sre
NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
grafana                             1/1     1            1           3m41s
prometheus-kube-state-metrics       1/1     1            1           3m52s
prometheus-prometheus-pushgateway   1/1     1            1           3m52s
prometheus-server                   1/1     1            1           3m52s
upcommerce-app-two                  0/1     1            0           17s
```

### Now list the pods
```
kubectl get pods -n sre
NAME                                                READY   STATUS    RESTARTS   AGE
grafana-596df764cb-n8sh8                            1/1     Running   0          5m36s
prometheus-alertmanager-0                           1/1     Running   0          5m47s
prometheus-kube-state-metrics-65468947fb-57sz5      1/1     Running   0          5m47s
prometheus-prometheus-node-exporter-vtnnc           1/1     Running   0          5m47s
prometheus-prometheus-pushgateway-76976dc66-kjkjs   1/1     Running   0          5m47s
prometheus-server-8444b5b7f7-2lxsw                  2/2     Running   0          5m47s
upcommerce-app-two-56cff9c64d-fsbrc                 0/1     Pending   0          2m12s
```

### Run this command to describe the pod that shows as pending 
```
kubectl describe pod upcommerce-app-two-65745f9d8b-j9scl -n sre


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

## Root Cause

### Check minikube allocatable CPUs.
```
kubectl get node minikube -o yaml | grep allocatable -A2
```
<img width="535" alt="image" src="https://github.com/aishwaryasarath/sre-week-two/assets/49971693/98d25ac3-60a6-497d-8cee-bdd278ca80bd">

Check the cpu limit in the deployment.yml
It shows it is setting a resource limit for cpu as 10.

This is what causes the below error in the deployment pod as there is insufficient cpu
```
0/1 nodes are available: 1 Insufficient cpu. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod..
```


## Fix 

### Current code of deployment has cpu limit as 10
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

### Change the cpu limit to 1 as below
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

### Apply the changes and check the status of the pod
```
kubectl apply -f deployment.yml -n sre
deployment.apps/upcommerce-app-two created
kubectl get pod -n sre
NAME                                                READY   STATUS    RESTARTS   AGE
grafana-596df764cb-n8sh8                            1/1     Running   0          12m
prometheus-alertmanager-0                           1/1     Running   0          12m
prometheus-kube-state-metrics-65468947fb-57sz5      1/1     Running   0          12m
prometheus-prometheus-node-exporter-vtnnc           1/1     Running   0          12m
prometheus-prometheus-pushgateway-76976dc66-kjkjs   1/1     Running   0          12m
prometheus-server-8444b5b7f7-2lxsw                  2/2     Running   0          12m
upcommerce-app-two-846fd54f5d-rn9x4                 1/1     Running   0          4s
```
