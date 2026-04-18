# Конфигурация в переменных среды (Envvar Configuration)

<br/>

## Разбор примеров из книги

<br/>

```shell
$ minikube start
```

<br/>

```shell
$ kubectl create configmap random-generator-config \
 --from-literal=PATTERN=EnvVarConfiguration \
 --from-literal="ILLEG.AL=Invalid Envvar name"
```

<br/>

```shell
$ kubectl create secret generic random-generator-secret --from-literal=seed=11232156346
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Example Pod to demonstrate the usage of environment variables
---
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
      # Import all entries of a ConfigMap as env vars, and add a prefix
      # RANDOM_ to every var
      env:
        # Literal environment variable
        - name: LOG_FILE
          value: /tmp/random.log
        # Pick up configuration from a secret
        - name: SEED
          valueFrom:
            secretKeyRef:
              name: random-generator-secret
              key: seed
        # Pick up all configuration from a config map
        - name: PORT
          value: '8181'
        # Get the Pod's IP address via the Downward API
        - name: IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        # Build an URL from already defined variables. $(CONTEXT) is not defined yet
        # and will be left unresolved
        - name: MY_URL
          value: 'https://$(IP):$(PORT)/$(CONTEXT)'
        # Refer to an env var RANDOM_PATTERN from the secret imported above
        - name: DESCRIPTION
          value: 'Welcome to $(RANDOM_PATTERN) !'
        # Path is defined here, but too late for being resolved in MY_URL
        - name: CONTEXT
          value: 'login/'
      envFrom:
        - configMapRef:
            name: random-generator-config
          prefix: RANDOM_
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
  selector:
    app: random-generator
  ports:
    - name: random
      port: 8080
      protocol: TCP
      targetPort: 8080
  type: NodePort
EOF
```

<br/>

```shell
$ minikube_ip=$(minikube ip)
$ port=$(kubectl get svc random-generator -o jsonpath='{.spec.ports[0].nodePort}')
```

<br/>

```shell
$ curl ${minikube_ip}:${port}/info | jq .
{
  "memory.free": 98,
  "seed": 11232156346,
  "memory.used": 154,
  "cpu.procs": 4,
  "memory.max": 3964,
  "logFile": "/tmp/random.log",
  "pattern": "EnvVarConfiguration",
  "id": "8f2047bf-7fbf-498c-82ad-b92ccb45e6f4",
  "env": {
    "PATH": "/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "RANDOM_GENERATOR_PORT_8080_TCP_PORT": "8080",
    "KUBERNETES_PORT": "tcp://10.96.0.1:443",
    "JAVA_HOME": "/opt/java/openjdk",
    "MY_URL": "https://10.244.0.3:8181/$(CONTEXT)",
    "RANDOM_GENERATOR_SERVICE_HOST": "10.110.36.17",
    "LANG": "en_US.UTF-8",
    "KUBERNETES_SERVICE_HOST": "10.96.0.1",
    "CONTEXT": "login/",
    "DESCRIPTION": "Welcome to EnvVarConfiguration !",
    "RANDOM_PATTERN": "EnvVarConfiguration",
    "SEED": "11232156346",
    "RANDOM_GENERATOR_SERVICE_PORT_RANDOM": "8080",
    "JAVA_VERSION": "jdk-17.0.9+9",
    "RANDOM_GENERATOR_PORT_8080_TCP_ADDR": "10.110.36.17",
    "LOG_FILE": "/tmp/random.log",
    "LANGUAGE": "en_US:en",
    "KUBERNETES_PORT_443_TCP": "tcp://10.96.0.1:443",
    "KUBERNETES_PORT_443_TCP_ADDR": "10.96.0.1",
    "PORT": "8181",
    "IP": "10.244.0.3",
    "RANDOM_GENERATOR_SERVICE_PORT": "8080",
    "KUBERNETES_PORT_443_TCP_PROTO": "tcp",
    "KUBERNETES_SERVICE_PORT": "443",
    "RANDOM_GENERATOR_PORT_8080_TCP": "tcp://10.110.36.17:8080",
    "RANDOM_GENERATOR_PORT_8080_TCP_PROTO": "tcp",
    "HOSTNAME": "random-generator",
    "LC_ALL": "en_US.UTF-8",
    "KUBERNETES_PORT_443_TCP_PORT": "443",
    "RANDOM_ILLEG.AL": "Invalid Envvar name",
    "RANDOM_GENERATOR_PORT": "tcp://10.110.36.17:8080",
    "KUBERNETES_SERVICE_PORT_HTTPS": "443",
    "HOME": "/"
  },
  "version": "1.0"
}
```

<br/>

```shell
$ minikube stop && minikube delete
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
