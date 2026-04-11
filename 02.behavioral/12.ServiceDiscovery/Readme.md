# Обнаружение сервисов (Service Discovery)

<br/>

<img src="../../img/chapter12-pic01.png">

<br/>

<img src="../../img/chapter12-pic02.png">

<br/>

### Разбор примеров из книги

<br/>

<br/>

```shell
$ minikube start --mount --mount-string="$(pwd)/logs:/tmp/example"
```

<br/>

```shell
$ minikube addons enable ingress
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# DeploymentConfig for starting up the random-generator
apiVersion: apps/v1
kind: Deployment
metadata:
  name: random-generator
spec:
  replicas: 4
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
EOF
```

<br/>

### Cluster-internal Service

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
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
EOF
```

<br/>

```shell
$ kubectl get svc
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP    4m22s
random-generator   ClusterIP   10.97.116.136   <none>        8080/TCP   7s
```

<br/>

```shell
$ kubectl run dbg --image=k8spatterns/curl-jq --command -- sleep infinity && \
  kubectl wait --for=condition=Ready pod/dbg && \
  kubectl exec -it dbg -- ash
```

<br/>

```shell
// Check DNS entry
# dig random-generator.default.svc.cluster.local
```

<br/>

```shell
/ # env | grep RANDOM
RANDOM_GENERATOR_PORT_8080_TCP_ADDR=10.97.116.136
RANDOM_GENERATOR_SERVICE_HOST=10.97.116.136
RANDOM_GENERATOR_PORT_8080_TCP_PORT=8080
RANDOM_GENERATOR_PORT_8080_TCP_PROTO=tcp
RANDOM_GENERATOR_SERVICE_PORT=8080
RANDOM_GENERATOR_PORT=tcp://10.97.116.136:8080
RANDOM_GENERATOR_PORT_8080_TCP=tcp://10.97.116.136:8080
```

<br/>

```shell
/ # dig random-generator.default.svc.cluster.local

***
;; ANSWER SECTION:
random-generator.default.svc.cluster.local. 30 IN A 10.97.116.136
***
```

<br/>

```shell
/ # exit
```

<br/>

### Service with type NodePort

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: random-generator
spec:
  type: NodePort
  selector:
    app: random-generator
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
      nodePort: 32766
EOF
```

<br/>

```shell
$ kubectl get svc random-generator
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
random-generator   NodePort   10.97.116.136   <none>        8080:32766/TCP   11m
```

<br/>

```shell
$ minikube ip
192.168.58.2
```

<br/>

```shell
$ curl -s http://192.168.58.2:32766 | jq .
{
  "random": -55177483,
  "id": "2ca94a03-e277-4a69-a389-878d5cf5ce4f",
  "version": "1.0"
}
```

<br/>

### Service with type LoadBalancer

```shell
$ minikube addons enable metallb
```

<br/>

```shell
$ minikube ip
192.168.58.2
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.58.20-192.168.58.30
EOF
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: random-generator
spec:
  type: LoadBalancer
  selector:
    app: random-generator
  ports:
    - port: 8080
      targetPort: 8080
      protocol: TCP
EOF
```

<br/>

```shell
$ kubectl get svc
NAME               TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
kubernetes         ClusterIP      10.96.0.1       <none>          443/TCP          20m
random-generator   LoadBalancer   10.97.116.136   192.168.58.20   8080:32766/TCP   16m
```

<br/>

```shell
// Pick port from the service definition and curl
$ ip=$(kubectl get svc random-generator -o jsonpath={.status.loadBalancer.ingress[0].ip})
```

<br/>

```shell
$ echo $ip
192.168.58.20
```

<br/>

```shell
$ curl -s http://$ip:8080 | jq .
{
  "random": 1291076490,
  "id": "04e61c73-b749-45ea-8da6-1417e8636bfe",
  "version": "1.0"
}
```

<br/>

### Ingress

<br/>

```
$ kubectl delete service random-generator
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
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
EOF
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: random-generator
spec:
  defaultBackend:
    service:
      name: random-generator
      port:
        number: 8080
EOF
```

<br/>

```shell
// Нужно подождать
$ kubectl get ingress
NAME               CLASS   HOSTS   ADDRESS        PORTS   AGE
random-generator   nginx   *       192.168.58.2   80      3m19s
```

<br/>

```shell
$ minikube ip
192.168.58.2
```

<br/>

```shell
$ curl -s http://192.168.58.2/  | jq .
{
  "random": -1868244448,
  "id": "3877c2c6-bb22-40b9-aa57-bb5fa95e7d84",
  "version": "1.0"
}
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
