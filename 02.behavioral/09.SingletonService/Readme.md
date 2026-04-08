# Сервис-одиночка (Singleton Service)

<br/>

```shell
$ kind create cluster --config kind-multinode.yml
```

<br/>

```shell
$ kubectl get nodes
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   26s   v1.34.0
kind-worker          Ready    <none>          15s   v1.34.0
kind-worker2         Ready    <none>          15s   v1.34.0
```

<br/>

**Now let’s create a Deployment with six Pods:**

```yaml
$ cat << EOF | kubectl apply -f -
# Deployment for random-generator service for starting up the random-generator
apiVersion: apps/v1
kind: Deployment
metadata:
  name: random-generator
spec:
  replicas: 6
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: random-generator
  template:
    metadata:
      labels:
        app: random-generator
    spec:
      containers:
        - image: k8spatterns/random-generator:1.0
          name: random-generator
          env:
            # Tell random-generator to wait 5 seconds when starting up
            - name: DELAY_STARTUP
              value: '5'
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 20
          readinessProbe:
            # We are checking for a file created by our app when its ready
            exec:
              command: ['stat', '/tmp/random-generator-ready']
          ports:
            - containerPort: 8080
              protocol: TCP
      # Allow scheduling also on the master nodes, which typically is tainted
      # for no-schedule
      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
EOF
```

<br/>

```shell
$ kubectl get pods -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
NAME                               NODE
random-generator-f4d97b74d-5nr94   kind-worker2
random-generator-f4d97b74d-68grl   kind-worker
random-generator-f4d97b74d-7hpl2   kind-worker
random-generator-f4d97b74d-88dgr   kind-worker2
random-generator-f4d97b74d-n5mbx   kind-worker
random-generator-f4d97b74d-thgk4   kind-worker2
```

<br/>

For being sure that always four Pods are running, we create a PodDisruptionBudget with

```yaml
$ cat << EOF | kubectl apply -f -
# Pod disruption budget
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: random-generator-pdb
spec:
  selector:
    matchLabels:
      # Used for counting active Pods for which this PDB is used
      app: random-generator
  # At least four need to be available all the time
  minAvailable: 4
EOF
```

<br/>

```shell
$ kubectl get PodDisruptionBudget
NAME                   MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
random-generator-pdb   4               N/A               2                     16s
```

<br/>

```shell
$ kubectl drain --ignore-daemonsets kind-worker >/dev/null 2>&1 &
```

<br/>

```shell
$ kubectl get nodes
NAME                 STATUS                     ROLES           AGE     VERSION
kind-control-plane   Ready                      control-plane   6m59s   v1.34.0
kind-worker          Ready,SchedulingDisabled   <none>          6m48s   v1.34.0
kind-worker2         Ready                      <none>          6m48s   v1.34.0
```

<br/>

```shell
$ watch kubectl get pods -o=custom-columns=NAME:.metadata.name,NODE:.spec.nodeName
NAME                               NODE
random-generator-f4d97b74d-5nr94   kind-worker2
random-generator-f4d97b74d-5pc85   kind-worker2
random-generator-f4d97b74d-88dgr   kind-worker2
random-generator-f4d97b74d-9bww4   kind-worker2
random-generator-f4d97b74d-c2zfh   kind-worker2
random-generator-f4d97b74d-thgk4   kind-worker2
```

You can undo the drain operation with

```shell
$ kubectl uncordon kind-worker
```

And restore the Deployment with

```shell
$ kubectl scale deployment random-generator --replicas 0
$ kubectl scale deployment random-generator --replicas 6
```

<br/>

```shell
$ kind delete cluster
```
