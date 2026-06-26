# Deploying the ToDo App to Kubernetes

Clone the repo and apply the manifests in order:

```bash
git clone https://github.com/maxmlv/devops_todolist_kubernetes_task_5_working_with_deployments.git
cd devops_todolist_kubernetes_task_5_working_with_deployments
```

## 1. Create the cluster

```bash
kind create cluster --config cluster.yml
```

```bash
kubectl config use-context kind-kind
```

---

## 2. Deploy

```bash
kubectl apply -f .infrastructure/namespace.yml
kubectl apply -f .infrastructure/metrics-server.yml
kubectl apply -f .infrastructure/deployment.yml
kubectl apply -f .infrastructure/cluster-ip.yml
kubectl apply -f .infrastructure/node-port.yml
kubectl apply -f .infrastructure/hpa.yml
```

Verify everything came up:

```bash
kubectl get pods -n mateapp
kubectl get deployments -n mateapp
kubectl get svc -n mateapp
kubectl get hpa -n mateapp
```

```
Pods:
            NAME                       READY   STATUS    RESTARTS   AGE
            todoapp-84b75c675b-fn6bw   1/1     Running   0          18m
            todoapp-84b75c675b-rdgwf   1/1     Running   0          18m

Deploy:
            NAME      READY   UP-TO-DATE   AVAILABLE   AGE
            todoapp   2/2     2            2           77m

Services:
            NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
            todoapp-clusterip   ClusterIP   10.96.133.244   <none>        80/TCP         70m
            todoapp-node-port   NodePort    10.96.132.71    <none>        80:30007/TCP   70m

HPA:
            NAME          REFERENCE            TARGETS                         MINPODS   MAXPODS   REPLICAS   AGE
            todoapp-hpa   Deployment/todoapp   cpu: 17%/70%, memory: 49%/70%   2         5         2          21m
```

---

## 3. Resource requests and limits

```yaml
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
```

**Why these values:** the app is a lightweight, single-threaded Django dev server doing simple CRUD operations — no heavy computation, no caching layer. Measured idle usage (`kubectl top pods`) came in around 17-18m CPU and ~62Mi memory per pod. `requests` are set above that baseline with real headroom (so normal idle fluctuation doesn't read as "near capacity" to the HPA), and `limits` are set roughly 2x `requests` to allow for occasional spikes without inviting unbounded growth.

```bash
kubectl top pods -n mateapp
```

```
NAME                       CPU(cores)   MEMORY(bytes)   
todoapp-84b75c675b-fn6bw   18m          63Mi            
todoapp-84b75c675b-rdgwf   18m          62Mi 
```

---

## 4. HPA configuration

```yaml
spec:
  minReplicas: 2
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
```

**Why these values:** `minReplicas: 2` keeps the app available even if one pod is mid-restart or being rescheduled. `maxReplicas: 5` caps scale-out at a level the cluster can comfortably support without real load-testing data to justify going higher. `70%` on both CPU and memory leaves comfortable headroom above measured idle usage (~16-18% CPU, ~48% memory against the `requests` above) so the app scales in response to genuine increased load, not idle-state noise.

```bash
kubectl get hpa -n mateapp
```

```
NAME          REFERENCE            TARGETS                         MINPODS   MAXPODS   REPLICAS   AGE
todoapp-hpa   Deployment/todoapp   cpu: 18%/70%, memory: 49%/70%   2         5         2          25m
```

---

## 5. Deployment strategy

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  replicas: 2
```

**Why these values:** `maxUnavailable: 1` and `maxSurge: 1` allow one pod to be replaced at a time during an update — at minimum 2 replicas, this means at least 1 pod stays available throughout the rollout, with at most one extra pod temporarily created to ease the transition. This keeps updates safe without requiring extra cluster capacity.

---

## 6. Accessing the app

### NodePort

Application is available at `http://localhost:30007` via the NodePort service.

### ClusterIP with port-forward

```bash
kubectl port-forward service/todoapp-clusterip 8081:80 -n mateapp
```

```
Forwarding from 127.0.0.1:8081 -> 8080
Forwarding from [::1]:8081 -> 8080
```

Application is available at `http://localhost:8081`

### Access from Busybox container

```bash
kubectl apply -f .infrastructure/busybox.yml
kubectl exec -it busybox -n mateapp -- sh
```

```bash
curl http://todoapp-clusterip.mateapp.svc.cluster.local
```