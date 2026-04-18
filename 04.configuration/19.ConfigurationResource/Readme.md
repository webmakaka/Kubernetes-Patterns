# Конфигурация в ресурсах (Configuration Resource)

<br/>

## Разбор примеров из книги

<br/>

```shell
$ minikube start
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: random-generator-config
data:
  PATTERN: Configuration Resource
  application.properties: |
    # Random Generator config
    log.file=/tmp/generator.log
    version=1.0
    server.port=7070
  # Used with a prefix 'env.' for being picked up
  # by a Pod's 'envFrom' declaration:
  EXTRA_OPTIONS: 'high-secure,native'
  SEED: '432576345'
# Make this config map immutable
immutable: true
EOF
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Environment variables picked up from a ConfigMap
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
      env:
        # Configure Spring boot to pick up that configuration coming from
        # a ConfigMap
        - name: SPRING_CONFIG_LOCATION
          value: /config/app/random-generator.properties
        # Pattern environment variable is picked up from a config map
        - name: PATTERN
          valueFrom:
            configMapKeyRef:
              name: random-generator-config
              key: PATTERN
      # Use 'envFrom' to pick up multiple enviroment variables from a config map
      envFrom:
        - configMapRef:
            name: random-generator-config
            optional: false
          prefix: CONFIG_
      volumeMounts:
        - name: config-volume
          mountPath: /config
  volumes:
    # Volume is mounted directly from a config map
    - name: config-volume
      configMap:
        name: random-generator-config
        items:
          # Mount only application properties under the provided path
          # and with the given permission
          - key: application.properties
            path: app/random-generator.properties
            mode: 0444
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
      targetPort: 7070
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
  "memory.free": 101,
  "memory.used": 154,
  "cpu.procs": 4,
  "memory.max": 3964,
  "logFile": "/tmp/generator.log",
  "pattern": "Configuration Resource",
  "id": "1e8387d8-314e-4c07-afaa-8cc11b536414",
  "env": {
    "LANGUAGE": "en_US:en",
    "KUBERNETES_PORT_443_TCP": "tcp://10.96.0.1:443",
    "PATH": "/opt/java/openjdk/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "KUBERNETES_PORT_443_TCP_ADDR": "10.96.0.1",
    "KUBERNETES_PORT": "tcp://10.96.0.1:443",
    "JAVA_HOME": "/opt/java/openjdk",
    "CONFIG_application.properties": "# Random Generator config\nlog.file=/tmp/generator.log\nversion=1.0\nserver.port=7070\n",
    "KUBERNETES_PORT_443_TCP_PROTO": "tcp",
    "LANG": "en_US.UTF-8",
    "KUBERNETES_SERVICE_HOST": "10.96.0.1",
    "KUBERNETES_SERVICE_PORT": "443",
    "CONFIG_SEED": "432576345",
    "SPRING_CONFIG_LOCATION": "/config/app/random-generator.properties",
    "PATTERN": "Configuration Resource",
    "HOSTNAME": "random-generator",
    "LC_ALL": "en_US.UTF-8",
    "CONFIG_PATTERN": "Configuration Resource",
    "KUBERNETES_PORT_443_TCP_PORT": "443",
    "CONFIG_EXTRA_OPTIONS": "high-secure,native",
    "JAVA_VERSION": "jdk-17.0.9+9",
    "KUBERNETES_SERVICE_PORT_HTTPS": "443",
    "HOME": "/"
  },
  "version": "1.0"
}
```

<br/>

```shell
$ curl ${minikube_ip}:${port}/info  | \
   jq '.env | with_entries(select(.key | startswith("CONFIG_")))'
{
  "CONFIG_application.properties": "# Random Generator config\nlog.file=/tmp/generator.log\nversion=1.0\nserver.port=7070\n",
  "CONFIG_SEED": "432576345",
  "CONFIG_PATTERN": "Configuration Resource",
  "CONFIG_EXTRA_OPTIONS": "high-secure,native"
}
```

<br/>

```shell
$ kubectl exec random-generator -- ls -l /config/app/random-generator.properties
-r--r--r-- 1 root root 83 Apr 17 01:23 /config/app/random-generator.properties
```

<br/>

```shell
$ kubectl patch configmap random-generator-config --type merge -p '{"data":{"newKey":"newValue"}}'
The ConfigMap "random-generator-config" is invalid: data: Forbidden: field is immutable when `immutable` is set
```

<br/>

```shell
$ minikube stop && minikube delete
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
