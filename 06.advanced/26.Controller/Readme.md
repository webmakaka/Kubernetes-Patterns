# Контроллер (Controller)

Контроллер реализует активный процесс согласования, основанный на наблюдении за состоянием объектов. Он анализирует фактическое и желаемое состояния и посылает инструкции, стремясь привести текущее состояние окружения к желаемому. Kubernetes использует этот механизм во множестве своих внутренних контроллеров, и его же можно переиспользовать в пользовательских контроллерах.

Этот контроллер в виде скрипта командной оболочки, конечно, нельзя использовать в рабочей среде (например, потому что цикл обработки событий может
остановиться в любой момент), но он хорошо раскрывает основные идеи на небольшом объеме кода.

<br/>

## Разбор примеров из книги

<br/>

Этот пример демонстрирует работу простого контроллера, который отслеживает изменения в ConfigMaps и перезапускает связанные с ними поды.
Данный контроллер следит за всеми ConfigMap в том пространстве имен (namespace), где он развернут, и реагирует на аннотацию k8spatterns.com/podDeleteSelector. Если эта аннотация присутствует в изменившемся ConfigMap, ее значение используется как селектор меток (label selector) для поиска подов, которые нужно удалить. Предполагается, что эти поды управляются фоновым контроллером (например, Deployment), поэтому они будут автоматически созданы заново и смогут подхватить обновленную конфигурацию.

<br/>

## Пример 1

<br/>

```shell
$ minikube start
```

<br/>

```shell
$ kubectl create configmap config-watcher-controller --from-file=./config-watcher-controller.sh
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Service account required for watching to resources
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-watcher-controller
---
# Bind to 'edit' role to allow for watching resources and restarting pods
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-watcher-controller
subjects:
  - kind: ServiceAccount
    name: config-watcher-controller
roleRef:
  name: edit
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
---
# Controller with kubeapi-proxy sidecar for easy access to the API server
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: k8spatterns
    pattern: Controller
    role: contoller
  name: config-watcher-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-watcher-controller
  template:
    metadata:
      labels:
        project: k8spatterns
        pattern: Controller
        app: config-watcher-controller
        role: controller
    spec:
      # A serviceaccount is needed to watch events
      # and to allow for restarting pods. For now its
      # associated with the 'edit' role
      serviceAccountName: config-watcher-controller
      containers:
        - name: kubeapi-proxy
          image: k8spatterns/kubeapi-proxy
        - name: config-watcher
          image: k8spatterns/curl-jq
          env:
            # The operator watches the namespace in which the controller
            # itself is installed (by using the Downward API)
            - name: WATCH_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          command:
            - 'sh'
            - '/watcher/config-watcher-controller.sh'
          # Mount script from a config map for ease of change
          volumeMounts:
            - mountPath: '/watcher'
              name: config-watcher-controller
      volumes:
        # Volume holding the controller script
        - name: config-watcher-controller
          configMap:
            name: config-watcher-controller
EOF
```

<br/>

```shell
$ kubectl logs -f $(kubectl get pods -l role=controller -o name) config-watcher

:: Starting to wait for events
::: ADDED -- config-watcher-controller --
::: ADDED -- kube-root-ca.crt --
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# ConfigMap which holds the value of the data to serve by
# the webapp
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  annotations:
    k8spatterns.com/podDeleteSelector: 'app=webapp'
data:
  message: 'Welcome to Kubernetes Patterns !'
---
# Deployment for a super simple HTTP server which
# serves the value of an environment variable to the browser.
# The env-var is picked up from a config map
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: app
          image: k8spatterns/mini-http-server
          ports:
            - containerPort: 8080
          env:
            # Message to print is taken from the ConfigMap as env var.
            # Note that changes to the ConfigMap require a restart of the Pod
            - name: MESSAGE
              valueFrom:
                configMapKeyRef:
                  name: webapp-config
                  key: message
---
# Service for accessing the web server via port 8080
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  ports:
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 8080
  type: NodePort
  selector:
    app: webapp
EOF
```

<br/>

```shell
// OK!
$ minikube service webapp
```

<br/>

Я патчу configmap и pod рестартится

```shell
// OK!
$ kubectl patch configmap webapp-config -p '{"data":{"message":"Take this update!"}}'
```

<br/>

```shell
// OK!
$ minikube service webapp
```

<br/>

```shell
$ kubectl logs -f $(kubectl get pods -l role=controller -o name) config-watcher
::: Starting to wait for events
::: ADDED -- config-watcher-controller --
::: ADDED -- kube-root-ca.crt --
::: ADDED -- webapp-config -- app%3Dwebapp
::: MODIFIED -- webapp-config -- app%3Dwebapp
::::: Deleting pods with app%3Dwebapp
::::: Deleted pod webapp-75b495b7bd-8f8p2
```

<br/>

```shell
$ minikube stop && minikube delete
```

<br/>

## Пример 2

<br/>

Этот пример демонстрирует работу простого контроллера, который реагирует на определенную аннотацию в сервисах (Services).

Данный контроллер отслеживает через Kubernetes API появление новых сервисов: если у сервиса есть аннотация expose=/path, контроллер создает для него объект Ingress, соответствующий указанному пути. Если Ingress для этого сервиса уже существует, никаких действий не предпринимается. Также стоит учесть, что Ingress не удаляется автоматически при удалении самого сервиса.

<br/>

```shell
$ minikube start
$ minikube addons enable ingress
```

<br/>

```shell
$ cd expose-controller/
$ kubectl create configmap expose-controller-script --from-file=./expose-controller.sh
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Service account required for watching to resources
apiVersion: v1
kind: ServiceAccount
metadata:
  name: expose-controller
---
# Bind to an appropriate permission
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: expose-controller
subjects:
  - kind: ServiceAccount
    name: expose-controller
roleRef:
  name: edit
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
---
# Example Deployment using a config map as input for a template
# which is processed from an init-container
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: k8spatterns
    pattern: Controller
  name: expose-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: expose-controller
  template:
    metadata:
      labels:
        project: k8spatterns
        pattern: Controller
        app: expose-controller
    spec:
      serviceAccountName: expose-controller
      containers:
        - name: kubeapi-proxy
          image: k8spatterns/kubeapi-proxy
        - name: expose-controller
          image: k8spatterns/curl-jq
          env:
            # The operator watches the namespace in which the controller
            # itself is installed (by using the Downward API)
            - name: WATCH_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          command:
            - 'sh'
            - '/expose-script/expose-controller.sh'
          volumeMounts:
            - mountPath: '/expose-script'
              name: expose-controller-script
      volumes:
        - name: expose-controller-script
          configMap:
            name: expose-controller-script
EOF
```

<br/>

```shell
$ kubectl get ingress
No resources found in default namespace.
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Example for a HTTP server with a git pulling sidecar
---
apiVersion: v1
kind: List
items:
  - apiVersion: v1
    kind: Pod
    metadata:
      name: web-app
      labels:
        project: k8spatterns
        pattern: Controller
        app: web-app
    spec:
      containers:
        # Main container is a stock httpd serving from /var/www/html
        - name: app
          image: centos/httpd
          ports:
            - containerPort: 80
          volumeMounts:
            - mountPath: /var/www/html/mdn
              name: git
        # Sidecar poll every 10 minutes a given repository with git
        - name: poll
          image: bitnami/git
          volumeMounts:
            - mountPath: /var/lib/data
              name: git
          env:
            - name: GIT_REPO
              value: https://github.com/mdn/beginner-html-site-scripted
          command:
            - 'sh'
            - '-c'
            - |
              if [ ! -d .git ]; then git clone $(GIT_REPO) .; fi
              while true; do git pull; sleep 600; done
          workingDir: /var/lib/data
      volumes:
        # The shared directory for holding the files
        - emptyDir: {}
          name: git
  # A service which opens a NodePort is added for your convenience
  # but is not necessarily required for this example:
  - apiVersion: v1
    kind: Service
    metadata:
      labels:
        project: k8spatterns
        pattern: Controller
        app: web-app
      annotations:
        # Annotation indicating that this path should be reachable as
        # ingress route and will be picked up by the ingress controller:
        expose: '/mdn'
      name: web-app
    spec:
      ports:
        - name: http
          port: 8080
          protocol: TCP
          targetPort: 80
      selector:
        app: web-app
EOF
```

<br/>

```shell
$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
expose-controller-74d596c8d-7nxcm   2/2     Running   0          24m
web-app                             2/2     Running   0          23s
```

<br/>

Создался ingress

```shell
$ kubectl get ingress
NAME      CLASS   HOSTS   ADDRESS   PORTS   AGE
web-app   nginx   *                 80      14s
```

<br/>

```shell
$ minikube ip
192.168.49.2
```

<br/>

```
// OK!
https://192.168.49.2/mdn
```

<br/>

```shell
$ minikube stop && minikube delete
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
