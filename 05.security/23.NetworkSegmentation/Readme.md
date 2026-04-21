# Сегментация сети (NetworkSegmentation) - ограничение трафика, проходящего через под

<br/>

## Разбор примеров из книги

<br/>

```shell
$ minikube start --cni=calico
```

<br/>

```shell
$ kubectl get pods -n kube-system | grep calico
calico-kube-controllers-565c89d6df-6srdn   1/1     Running   0          5m33s
calico-node-cxnnq                          1/1     Running   0          5m33s
```

<br/>

### Network Policies

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
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
        has-metrics: 'true'
    spec:
      containers:
        - name: random-generator
          image: k8spatterns/random-generator:1.0
          ports:
            - containerPort: 8080
EOF
```

<br/>

```shell
$ kubectl run curl --image=curlimages/curl --restart=Never --command -- sleep infinity
```

<br/>

```shell
$ RANDOM_GENERATOR_POD_IP=$(kubectl get pod -l app=random-generator -o jsonpath='{.items[0].status.podIP}')
```

<br/>

```shell
$ echo $RANDOM_GENERATOR_POD_IP
10.244.120.67
```

<br/>

```shell
$ kubectl exec curl -- curl -s $RANDOM_GENERATOR_POD_IP:808 | jq .
{
  "random": -1796475887,
  "id": "da7c65c0-83c4-4768-a8a8-8bd6a3bacfea",
  "version": "1.0"
}
```

<br/>

**Deny All Policy**

Next, let’s create the NetworkPolicy for denying all incoming traffic by applying

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  ingress: []
EOF
```

<br/>

```shell
// The request should time out due to the deny-all policy.
$ kubectl exec curl -- curl -s $RANDOM_GENERATOR_POD_IP:8080
```

<br/>

**Allow Ingress from Pods**

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-access-to-random
spec:
  podSelector:
    # Policy applies to the random-generator app
    matchLabels:
      app: random-generator
  ingress:
    - from:
        - podSelector:
            # All Pods that carry "role: random-client" as label are allowed to access
            # our deployment
            matchLabels:
              role: random-client
      ports:
        - protocol: TCP
          port: 8080
EOF
```

<br/>

```shell
// TIMEOUT!
$ kubectl exec curl -- curl -s $RANDOM_GENERATOR_POD_IP:8080
```

<br/>

Now create another curl client, but this time with a label random-client that allows to pass the ingress NetworkPolicy

<br/>

```shell
$ kubectl run curl-random \
      --image=curlimages/curl \
      --labels=role=random-client \
      --restart=Never \
      --command -- sleep infinity
```

<br/>

Finally, Attempt to access the random generator service from the curl-access container:

<br/>

```shell
$ kubectl exec curl-random -- curl -s $RANDOM_GENERATOR_POD_IP:8080 | jq
{
  "random": -497917992,
  "id": "da7c65c0-83c4-4768-a8a8-8bd6a3bacfea",
  "version": "1.0"
}
```

<br/>

**Egress Policies**

Let’s continue our journey and restrict the egress access for our curl in the Pod random-client that we have created above.

For this, apply the following resource file that will only allows cluster-internal traffic, except for api.chucknorris.io (you might need to check the IP adresses in this resource file whether they are still pointing to api.chucknorris.io):

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-allow-internal-only
spec:
  policyTypes:
    # Type is needed here, otherwise it would also affect ingressesj
    - Egress
  podSelector: {}
  egress:
    # Only allow egress to all internal namespaces
    - to:
        - namespaceSelector: {}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-chucknorris
spec:
  policyTypes:
    - Egress
  # This rule applies to all Pods
  podSelector: {}
  egress:
    - to:
        # Add here all IP adresses for api.chucknorris.io or any other service
        # that you want allow
        - ipBlock:
            cidr: 104.21.41.162/32
        - ipBlock:
            cidr: 172.67.148.58/32
EOF
```

To verify, whether our egress policies work, let’s try the following three curl:

<br/>

```shell
$ kubectl exec curl-random -- curl -sm 5 $RANDOM_GENERATOR_POD_IP:8080 | jq .
{
  "random": -184040501,
  "id": "da7c65c0-83c4-4768-a8a8-8bd6a3bacfea",
  "version": "1.0"
}
```

<br/>

```shell
// command terminated with exit code 28
$ kubectl exec curl-random -- curl -sm 5 https://github.com
```

<br/>

```shell
// command terminated with exit code 28
$ kubectl exec curl-random -- curl -sm 5 https://api.chucknorris.io/jokes/random | jq .
```

<br/>

```shell
$ kubectl exec curl-random -- nslookup api.chucknorris.io
Server:		10.96.0.10
Address:	10.96.0.10:53

Non-authoritative answer:
Name:	api.chucknorris.io
Address: 172.67.159.75
Name:	api.chucknorris.io
Address: 104.21.9.74

Non-authoritative answer:
Name:	api.chucknorris.io
Address: 2606:4700:3036::6815:94a
Name:	api.chucknorris.io
Address: 2606:4700:3036::ac43:9f4b
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-chucknorris
spec:
  policyTypes:
    - Egress
  podSelector: {}
  egress:
    - to:
        - ipBlock:
            cidr: 172.67.159.75/32
        - ipBlock:
            cidr: 104.21.9.74/32
      ports:
        - protocol: TCP
          port: 443
EOF
```

```shell
$ kubectl exec curl-random -- curl -sm 5 https://api.chucknorris.io/jokes/random | jq .
{
  "categories": [],
  "created_at": "2020-01-05 13:42:28.984661",
  "icon_url": "https://api.chucknorris.io/img/avatar/chuck-norris.png",
  "id": "7V7754e1TGaCKNgeAWD7RQ",
  "updated_at": "2020-01-05 13:42:28.984661",
  "url": "https://api.chucknorris.io/jokes/7V7754e1TGaCKNgeAWD7RQ",
  "value": "Chuck Norris' daughter lost her virginity. . .he got it back"
}
```

<br/>

```shell
$ minikube stop && minikube delete
```

<br/>

### Authorization Policies

<br/>

```shell
$ minikube start --memory=8192 --cpus=4
```

<br/>

Next you need to install the lastest version of the istioctl binary.

<br/>

```shell
$ istioctl version
client version: 1.29.2
```

<br/>

```shell
$ istioctl install --set profile=demo
```

<br/>

```shell
$ kubectl label namespace default istio-injection=enabled
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
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
        has-metrics: 'true'
    spec:
      containers:
        - name: random-generator
          image: k8spatterns/random-generator:1.0
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: random-generator
spec:
  selector:
    app: random-generator
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF
```

<br/>

Now, create an AuthorizationPolicy resource that denies all traffic in all namespaces by default:

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: istio-system
spec: {}
EOF
```

<br/>

Next, create another AuthorizationPolicy that allows traffic to the /metrics endpoint for the random-generator application.

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-metrics
spec:
  # Match all pods that have a lbel "has-metrics
  selector:
    matchLabels:
      has-metrics: 'true'
  action: ALLOW
  rules:
    # Allow for all Pods that want to acess the "/actuator/health" endpoint
    - from:
        - source:
            namespaces: ['default']
      to:
        - operation:
            methods: ['GET']
            paths: ['/actuator/health']
EOF
```

<br/>

```shell
$ kubectl run curl --image=curlimages/curl --restart=Never --command -- sleep infinity
```

<br/>

```shell
$ kubectl exec curl -- curl -s http://random-generator/actuator/health  | jq
```

**response:**

```json
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 491068768256,
        "free": 15024513024,
        "threshold": 10485760,
        "exists": true
      }
    },
    "healthToggleIndicator": {
      "status": "UP",
      "details": {
        "toggle": true,
        "usedMemory": "47.0 MB",
        "totalMemory": "148.0 MB",
        "maxMemory": "3.9 GB",
        "freeMemory": "101.0 MB",
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
  "groups": ["liveness", "readiness"]
}
```

<br/>

```shell
// RBAC: access denied
$ kubectl exec curl -- curl -sm 5 http://random-generator/
```

<br/>

```shell
$ minikube stop && minikube delete
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
