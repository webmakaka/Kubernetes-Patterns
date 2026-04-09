# Проверка работоспособности (Health Probe)

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Deployment for starting up the random-generator with liveness and readiness probes
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
          env:
            # Tell random-generator to wait 20 seconds when starting up
            - name: DELAY_STARTUP
              value: '20'
          ports:
            - containerPort: 8080
              protocol: TCP
          livenessProbe:
            # Spring Boot's actuator comes in handy as a liveness probe check
            # You can use the endpoint "/toggle-heath" to toggle the health state
            httpGet:
              path: /actuator/health
              port: 8080
            # How long to wait until the liveness check should kick it.
            initialDelaySeconds: 30
          readinessProbe:
            # We are checking for a file created by our app when its ready
            exec:
              command: ['stat', '/tmp/random-generator-ready']
EOF
```

<br/>

```bash
$ kubectl get pods -w
```

<br/>

### Liveness Probes

<br/>

```bash
$ kubectl port-forward deployment/random-generator 8080:8080 2>&1
```

<br/>

```bash
// Check the liveness probe by querying the actuator
$ curl -s http://localhost:8080/actuator/health | jq .
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 491068768256,
        "free": 72061849600,
        "threshold": 10485760,
        "exists": true
      }
    },
    "healthToggleIndicator": {
      "status": "UP",
      "details": {
        "toggle": true,
        "usedMemory": "54.7 MB",
        "totalMemory": "152.0 MB",
        "maxMemory": "3.9 GB",
        "freeMemory": "97.3 MB",
        "availableProcessors": 4
      }
    },
    "livenessState": {
      "status": "UP"
    },
    "ping": {
      "status": "UP"
    },
    "readinessState": {
      "status": "UP"
    }
  },
  "groups": [
    "liveness",
    "readiness"
  ]
}
```

<br/>

```bash
// Toggle liveness check to off
$ curl -s http://localhost:8080/toggle-live
```

<br/>

```bash
// Recheck the liveness probe
$ curl -s http://localhost:8080/actuator/health | jq .
```

<br/>

```bash
$ kubectl get pods -w
```

<br/>

### Readiness Probes

<br/>

```bash
$ curl -s http://localhost:8080/toggle-ready
```

<br/>

Watch the pods for 1-2 mins

<br/>

```bash
$ kubectl get pods -w
```

<br/>

```bash
// Toggle readiness back on
$ curl -s http://localhost:8080/toggle-ready
```

<br/>

### Startup Probes

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Deployment for starting up the random-generator with liveness, readiness, and startup probes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: random-generator
spec:
  replicas: 1
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
        - image: k8spatterns/random-generator:2.0
          name: random-generator
          env:
            # Tell random-generator to wait 60 seconds when starting up so that the startup probe
            # can kick in earlier after 20s
            - name: DELAY_STARTUP
              value: '60'
          ports:
            - containerPort: 8080
              protocol: TCP
          livenessProbe:
            # Spring Boot's actuator comes in handy as a liveness probe check
            # You can use the endpoint "/toggle-heath" to toggle the health state
            httpGet:
              path: /actuator/health
              port: 8080
          readinessProbe:
            # We are checking for a file created by our app when its ready
            exec:
              command: ['stat', '/tmp/random-generator-ready']
          startupProbe:
            # Check the same endpoint as the liveness probe for the startup probe
            httpGet:
              path: /actuator/health
              port: 8080
            # Configure the startup probe to have a longer initial delay to allow the application to start
            initialDelaySeconds: 20
            # Number of attempts to perform the probe before considering it a failure
            failureThreshold: 30
            # Interval between probe attempts
            periodSeconds: 10
EOF
```

<br/>

```bash
$ kubectl get pods -w
```

<br/>

```bash
$ kubectl delete deployment random-generator
```

<br/>

### Readiness Gates

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Deployment for starting up the random-generator with liveness, readiness, and startup probes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: random-generator
spec:
  replicas: 1
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
          ports:
            - containerPort: 8080
              protocol: TCP
          readinessProbe:
            # We are checking for a file created by our app when its ready
            initialDelaySeconds: 20
            exec:
              command: ['stat', '/tmp/random-generator-ready']
      readinessGates:
        - conditionType: 'k8spatterns.com/RandomReady'
EOF
```

<br/>

```bash
$ kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE   IP            NODE              NOMINATED NODE   READINESS GATES
random-generator-69d966d6b7-srrch   1/1     Running   0          60s   10.244.0.31   marley-minikube   <none>           0/1
```

<br/>

```bash
$ pod=$(kubectl get pods -l app=random-generator -o name)
```

<br/>

```bash
$ kubectl get pod $pod -o jsonpath='{.spec.readinessGates}'
[{"conditionType":"k8spatterns.com/RandomReady"}]
```

<br/>

```bash
$ kubectl patch $pod --type='json' --subresource=status \
 -p='[{"op": "add",
       "path": "/status/conditions/-",
       "value": {"type": "k8spatterns.com/RandomReady", "status": "True"}}]'
```

<br/>

```bash
$ kubectl get pod -o wide
NAME                                READY   STATUS    RESTARTS   AGE     IP            NODE              NOMINATED NODE   READINESS GATES
random-generator-69d966d6b7-srrch   1/1     Running   0          2m38s   10.244.0.31   marley-minikube   <none>           1/1
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
