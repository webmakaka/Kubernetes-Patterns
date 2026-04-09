# Декларативное развертывание (Declarative Deployment)

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# A rolling update Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: random-generator
spec:
  # More than 1 replica is required for a rolling update
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      # Number of Pods which can be run temporarily in addition the replicas
      # specified during an updated
      # (so it could 4 in our case at max)
      maxSurge: 1
      # Number of Pods which can be unavaiable during the update. Here it could
      # be that only 2 Pods are running at a time during the update
      maxUnavailable: 1
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
            - name: PATTERN
              value: Declarative Deployment
          ports:
            - containerPort: 8080
              protocol: TCP
          # Readiness probes are very important for a RollingUpdate to work properly,
          # so don't forget them
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 15
          readinessProbe:
            exec:
              command: ['stat', '/tmp/random-generator-ready']
EOF
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Service object for accessing the example Deployment via Minikube's LoadBalancer
# Alternatively you could also use a "NodePort" service to access the service over a
# random IP or use Minikube's Ingress extension to acess the application
apiVersion: v1
kind: Service
metadata:
  name: random-generator
spec:
  selector:
    app: random-generator
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  type: LoadBalancer
EOF
```

<br/>

```bash
$ minikube --profile ${PROFILE} service random-generator --url > /tmp/random-url.txt
```

<br/>

```bash
$ url=$(cat /tmp/random-url.txt)
$ while true; do
  curl -s $url/info | jq '.version,.id'
  echo "==========================="
  sleep 1
done
```

<br/>

```
"1.0"
"9e479163-6c79-4f9a-bc7c-6690f0731d42"
===========================
"1.0"
"4d5e09af-4ff9-4393-aaf8-b8f001246552"
===========================
"1.0"
"b37c1f36-722a-46f3-b72c-a7c2dc1fdf11"
```

<br/>

```bash
$ kubectl set image deployment random-generator random-generator=k8spatterns/random-generator:2.0
```

<br/>

```
$ kubectl get pods -w
NAME                                READY   STATUS    RESTARTS   AGE
random-generator-79859cfc4b-9b844   1/1     Running   0          38s
random-generator-79859cfc4b-bzjvg   1/1     Running   0          38s
random-generator-79859cfc4b-fj7jl   0/1     Running   0          12s
random-generator-79859cfc4b-fj7jl   1/1     Running   0          13s
```

<br/>

```bash
$ kubectl rollout status deploy/random-generator
deployment "random-generator" successfully rolled out
```

<br/>

```
"1.0"
"b37c1f36-722a-46f3-b72c-a7c2dc1fdf11"
===========================
"1.0"
"b37c1f36-722a-46f3-b72c-a7c2dc1fdf11"
===========================
"1.0"
"b37c1f36-722a-46f3-b72c-a7c2dc1fdf11"
===========================
"2.0"
"395326f4-3b4a-4cb5-b2b3-22e5439a95d2"
===========================
"2.0"
"afe98ff5-24e9-41e8-acfe-d81cd2bf0326"
===========================
```

<br/>

```bash
# Rollback the Deployment
$ kubectl rollout undo deploy/random-generator
```

<br/>

```bash
# Check the update history
$ kubectl rollout history deploy/random-generator
deployment.apps/random-generator
REVISION  CHANGE-CAUSE
2         <none>
3         <none>

```

<br/>

**Finally, let’s switch the update strategy to Recreate:**

<br/>

```yaml
$ cat << 'EOF' | kubectl replace -f -
# A recreate (or fixed) Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: random-generator
spec:
  replicas: 3
  strategy:
    # Kill first all old Pods, then start the new version
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
            - name: PATTERN
              value: Declarative Deployment
          ports:
            - containerPort: 8080
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 15
          readinessProbe:
            exec:
              command: ['stat', '/tmp/random-generator-ready']
EOF
```

<br/>

```bash
$ kubectl get pods -w
```

<br/>

```bash
# Update to version 2.0 (or change to 1.0 when you have 2.0 running)
$ kubectl set image deployment random-generator random-generator=k8spatterns/random-generator:2.0
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
