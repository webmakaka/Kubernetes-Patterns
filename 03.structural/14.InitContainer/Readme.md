# Init-контейнеры (Init Container)

<br/>

### Разбор примеров из книги

<br/>

```shell
$ minikube start --mount --mount-string="$(pwd)/logs:/tmp/example"
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: www
  labels:
    app: www
spec:
  initContainers:
    - name: download
      image: bitnami/git
      # Clone an HTML page to be served
      command:
        - git
        - clone
        - https://github.com/mdn/beginner-html-site-scripted
        - /var/lib/data
      # Shared volume with main container
      volumeMounts:
        - mountPath: /var/lib/data
          name: source
  containers:
    # Simple static HTTP server for serving these pages
    - name: run
      image: docker.io/centos/httpd
      ports:
        - containerPort: 80
      # Shared volume with main container
      volumeMounts:
        - mountPath: /var/www/html
          name: source
  volumes:
    - emptyDir: {}
      name: source
EOF
```

<br/>

```shell
$ kubectl get pods -w
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: www
spec:
  selector:
    app: www
  ports:
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 80
  type: NodePort
EOF
```

<br/>

```shell
$ port=$(kubectl get svc www -o jsonpath='{.spec.ports[0].nodePort}')
```

<br/>

```shell
$ echo ${port}
30426
```

<br/>

```shell
$ minikube ip
192.168.58.2
```

<br/>

В логах ошибка: failed to create fsnotify watcher: too many open files

```shell
// Помогло!
// После перезагрузки сбрасываеются

$ sysctl fs.inotify.max_user_instances
fs.inotify.max_user_instances = 128

$ sysctl fs.inotify.max_user_watches
fs.inotify.max_user_watches = 65536

$ sudo sysctl fs.inotify.max_user_instances=8192
$ sudo sysctl fs.inotify.max_user_watches=524288
```

<br/>

**Пересоздаю pod:**

<br/>

```shell
$ kubectl delete pod www
```

<br/>

```
// OK!
http://192.168.58.2:30426
```

<br/>

```shell
$ kubectl delete pod www
$ kubectl delete service www
```

<br/>

# Init Container (random-generator example)

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
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
      initContainers:
        # Same image as main container, but used for calling the batch CLI
        - image: k8spatterns/random-generator:1.0
          name: init
          command:
            - java
            # Use / as classpath to pick up the class file
            - -classpath
            - /
            # Class running batch job
            - RandomRunner
            # 1. Arg: File to store data (on a PV)
            - /logs/random.log
            # 2. How many iterations
            - '100000'
          # Shared volume with main container. Use for initializing the batch file
          volumeMounts:
            - mountPath: /logs
              name: log-volume
      containers:
        # ------------------------------------------------
        # Main container sharing the /logs dir with the init container
        - image: k8spatterns/random-generator:1.0
          name: random-generator
          env:
            # The log file that we want to export
            - name: LOG_FILE
              value: /logs/random.log
          ports:
            # Application running on port 8080
            - containerPort: 8080
              protocol: TCP
          volumeMounts:
            - mountPath: /logs
              name: log-volume
      volumes:
        # New empty directory volume for sharing the log file between container
        - name: log-volume
          emptyDir: {}
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
$ kubectl get pods
```

<br/>

```shell
$ kubectl logs random-generator-574fdc8886-wr7jw -c init
Error: Could not find or load main class RandomRunner
Caused by: java.lang.ClassNotFoundException: RandomRunner
```

<br/>

**Меняю команду на:**

```yaml
***
command: ['find', '/', '-name', 'RandomRunner.class']
***
```

<br/>

**В логах вижу:**

```
│ find: ‘/proc/tty/driver’: Permission denied                                  │
│ /opt/RandomRunner.class                                                      │
│ find: ‘/root’: Permission denied                                             │
│ find: ‘/etc/ssl/private’: Permission denied                                  │
│ find: ‘/var/cache/apt/archives/partial’: Permission denied                   │
│ find: ‘/var/cache/ldconfig’: Permission denied
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
  replicas: 1
  selector:
    matchLabels:
      app: random-generator
  template:
    metadata:
      labels:
        app: random-generator
    spec:
      initContainers:
        # Same image as main container, but used for calling the batch CLI
        - image: k8spatterns/random-generator:1.0
          name: init
          command:
            - java
            - -classpath
            - /opt           # <--- Исправлено с / на /opt
            - RandomRunner
            - /logs/random.log
            - '100000'
          volumeMounts:
            - mountPath: /logs
              name: log-volume
      containers:
        # ------------------------------------------------
        # Main container sharing the /logs dir with the init container
        - image: k8spatterns/random-generator:1.0
          name: random-generator
          env:
            # The log file that we want to export
            - name: LOG_FILE
              value: /logs/random.log
          ports:
            # Application running on port 8080
            - containerPort: 8080
              protocol: TCP
          volumeMounts:
            - mountPath: /logs
              name: log-volume
      volumes:
        # New empty directory volume for sharing the log file between container
        - name: log-volume
          emptyDir: {}
EOF
```

<br/>

**После изменений норм запустился!**

<br/>

```shell
$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
random-generator-b687ddfc5-ngwc5   1/1     Running   0          45s
```

<br/>

```shell
$ port=$(kubectl get svc random-generator -o jsonpath={.spec.ports[0].nodePort})
```

<br/>

```shell
$ echo $port
```

<br/>

```shell
// OK!
$ curl -s http://192.168.58.2:$port/logs | head -10
```

<br/>

```shell
$ minikube stop
$ minikube delete
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
