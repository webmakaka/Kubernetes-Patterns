# Predictable Demands

<br/>

```bash
$ minikube start --mount --mount-string="$(pwd)/logs:/tmp/example" --memory 2G
```

<br/>

### Hard requirements

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
# Resource limits
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
  labels:
    app: random-generator
spec:
  containers:
  - image: k8spatterns/random-generator:1.0
    name: random-generator
    env:
    - name: PATTERN
      valueFrom:
        # First hard requirement for a config map to exist.
        configMapKeyRef:
          name: random-generator-config
          key: pattern
    # Enabling logging into the mounted volume
    - name: LOG_FILE
      value: /tmp/logs/random.log
    volumeMounts:
      - mountPath: /tmp/logs
        name: log-volume
  volumes:
    - name: log-volume
      # Second hard requirement is that the specified persitent volume claim
      # exists and is bound.
      persistentVolumeClaim:
        claimName: random-generator-log
EOF
```

<br/>

```bash
$ kubectl get pods
NAME               READY   STATUS    RESTARTS   AGE
random-generator   0/1     Pending   0          45s
```

<br/>

```bash
$ kubectl get pods
NAME               READY   STATUS    RESTARTS   AGE
random-generator   0/1     Pending   0          45s
```

<br/>

```bash
$ kubectl describe pod random-generator
***
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  90s   default-scheduler  0/1 nodes are available: persistentvolumeclaim "random-generator-log" not found. not found

```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
# Persistent volume mapping a hostPath. Works only on 1-node clusters like Minikube
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 10Mi
  # storageClassName must match between PV and PVC
  storageClassName: standard
  hostPath:
    # Mount by Minikube from local directory during 'minikube start'
    path: /tmp/example
---
# Persistent Volume Claim required by our random service
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: random-generator-log
spec:
  accessModes:
    - ReadWriteOnce
  # Must match the storageClassName of the PV
  storageClassName: standard
  resources:
    requests:
      storage: 10Mi
  volumeName: example
EOF
```

<br/>

```bash
$ kubectl describe pod random-generator
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  3m5s               default-scheduler  0/1 nodes are available: persistentvolumeclaim "random-generator-log" not found. not found
  Warning  FailedScheduling  10s                default-scheduler  0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims. not found
  Warning  FailedScheduling  10s (x2 over 10s)  default-scheduler  0/1 nodes are available: pod has unbound immediate PersistentVolumeClaims. not found
  Normal   Scheduled         10s                default-scheduler  Successfully assigned default/random-generator to marley-minikube
  Normal   Pulling           10s                kubelet            Pulling image "k8spatterns/random-generator:1.0"
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: random-generator-config
data:
  pattern: Predictable Demands
EOF
```

<br/>

```bash
$ kubectl get pods
NAME               READY   STATUS    RESTARTS   AGE
random-generator   1/1     Running   0          5m12s
```

<br/>

```bash
$ POD_IP=$(kubectl get pod -l app=random-generator -o jsonpath='{.items[0].status.podIP}')
```

<br/>

```bash
$ kubectl run -itq --rm --image=k8spatterns/curl-jq --restart=Never curl -- http://$POD_IP:8080 | jq .
```

<br/>

```bash
$ cat ./logs/random.log
```

<br/>

```bash
$ kubectl delete pod/random-generator
$ kubectl delete pvc random-generator-log
$ kubectl delete pv example
```

<br/>

### Resource limits

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
# DeploymentConfig for starting up the random-generator
apiVersion: apps/v1
kind: Deployment
metadata:
  name: random-generator
spec:
  replicas: 1
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
          ports:
            - containerPort: 8080
              protocol: TCP
          resources:
            # Initial resource request for CPU and memory
            requests:
              cpu: 100m
              memory: 100Mi
            # Upper limit until we want our application to grow at max
            limits:
              cpu: 200m
              memory: 200Mi
EOF
```

<br/>

```bash
$ kubectl logs -l app=random-generator -f | grep "=== "
2026-04-02 00:48:46.828  INFO 1 --- [           main] i.k.examples.RandomGeneratorApplication  : === Max Heap Memory:  96 MB
2026-04-02 00:48:46.829  INFO 1 --- [           main] i.k.examples.RandomGeneratorApplication  : === Used Heap Memory: 32 MB
2026-04-02 00:48:46.829  INFO 1 --- [           main] i.k.examples.RandomGeneratorApplication  : === Free Memory:      16 MB
2026-04-02 00:48:46.829  INFO 1 --- [           main] i.k.examples.RandomGeneratorApplication  : === Processors:       1
```

<br/>

```bash
$ patch=$(cat <<EOT
[
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/resources/requests/memory",
    "value": "30Mi"
  },
  {
    "op": "replace",
    "path": "/spec/template/spec/containers/0/resources/limits/memory",
    "value": "30Mi"
  }
]
EOT
)

$ kubectl patch deploy random-generator --type=json -p "$patch"
```

<br/>

```bash
$ kubectl get pods
NAME                                READY   STATUS      RESTARTS      AGE
random-generator-66fc69847b-snqcf   0/1     OOMKilled   4 (52s ago)   109s
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
