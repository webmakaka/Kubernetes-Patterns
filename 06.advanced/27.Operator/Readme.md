# Оператор (Operator)

<br/>

Паттерн Оператор (Operator) — это контроллер, который использует определения пользовательских ресурсов (CRD, Custom Resource Definition) для автоматизации рутинных операций по управлению конкретным приложением.

<br/>

CRD позволяют добавить в Kubernetes возможность управления понятиями из другой предметной области. CRD управляются так же, как любые другие ресурсы, через API Kubernetes, и хранятся во внутреннем хранилище etcd.

<br/>

Для разработки операторов доступно несколько наборов инструментов и фреймворков. Три основных проекта, помогающих в создании операторов:

- Kubebuilder, разработанный в рамках SIG API Machinery самого Kubernetes.
- Operator Framework, проект CNCF.
- Metacontroller из Google Cloud Platform.

<br/>

Прежде чем приступать к созданию своего оператора, внимательно изучите задачу, чтобы определить, соответствует ли она парадигме Kubernetes.
Во многих случаях вполне достаточно обычного контроллера, работающего со стандартными ресурсами. Его преимущество в том, что он не требует полномочий администратора кластера для регистрации CRD, хотя имеет ограничения, касающиеся безопасности и проверки.

<br/>

## Разбор примеров из книги

<br/>

Мы расширим пример из «Контроллер» и определим CRD типа ConfigWatcher.
Экземпляр этого CRD определяет ссылку на ресурс ConfigMap для наблюдения и то, какие поды должны перезапускаться при его изменении. Такой подход помогает устранить зависимость от ConfigMap в подах, так как нам не нужно изменять сами ресурсы ConfigMap и добавлять в них аннотации, отвечающие за запуск логики. Кроме того, в упрощенном примере контроллера, основанного на аннотациях, мы можем подключить ConfigMap только к одному приложению. С использованием CRD возможны произвольные комбинации ресурсов ConfigMap и подов.

<br/>

```shell
$ minikube start
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# CRD connecting a ConfigMap with a set of pods which needs to
# be restarted when the ConfigMap changes
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: configwatchers.k8spatterns.com
spec:
  scope: Namespaced
  group: k8spatterns.com
  names:
    # Kind of this CRD
    kind: ConfigWatcher
    # How to access them via client and REST api
    singular: configwatcher
    plural: configwatchers
    # How to access the CRDs as well (e.g. with "kubectl get cw")
    shortNames: [ cw ]
    # Adds Configwatcher to the "all" category (e.g. "kubectl get all")
    categories: [ all ]
  versions:
  - name: v1
    # Enabled
    served: true
    # The version stored in the backend
    storage: true
    # Validation schema
    schema:
      openAPIV3Schema:
        type: object
        properties:
          configMap:
            type: string
          podSelector:
            type: object
            additionalProperties:
              type: string
      openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              required:
                - configMap
                - podSelector
              properties:
                configMap:
                  type: string
                  description: Name of the ConfigMap to monitor for changes
                  minLength: 1
                podSelector:
                  type: object
                  description: Label selector used for selecting Pods
                  additionalProperties:
                    type: string
    # Additional columns to print when in kubectl get
    additionalPrinterColumns:
    - name: configmap
      description: Name of ConfigMap to watch
      type: string
      jsonPath: .spec.configMap
    - name: podselector
      description: Selector for Pods to restart
      type: string
      jsonPath: .spec.podSelector
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-watcher-crd
rules:
- apiGroups:
  - k8spatterns.com
  resources:
  - configwatchers
  - configwatchers/finalizers
  verbs: [ get, list, create, update, delete, deletecollection, watch ]
EOF
```

<br/>

```shell
$ kubectl get crd
NAME                             CREATED AT
configwatchers.k8spatterns.com   2026-04-25T08:49:41Z
```

<br/>

```shell
$ kubectl create configmap config-watcher-operator --from-file=./config-watcher-operator.sh
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Service account required for watching to resources
apiVersion: v1
kind: ServiceAccount
metadata:
  name: config-watcher-operator
---
# Bind to an appropriate permission
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-watcher:edit
subjects:
  - kind: ServiceAccount
    name: config-watcher-operator
roleRef:
  name: edit
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: config-watcher:crd
subjects:
  - kind: ServiceAccount
    name: config-watcher-operator
roleRef:
  name: config-watcher-crd
  kind: Role
  apiGroup: rbac.authorization.k8s.io
---
# Controller with kubeapi-proxy sidecar for easy access to the API server
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    project: k8spatterns
    pattern: Controller
  name: config-watcher-operator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: config-watcher-operator
  template:
    metadata:
      labels:
        project: k8spatterns
        pattern: Operator
        role: operator
        app: config-watcher-operator
    spec:
      serviceAccountName: config-watcher-operator
      containers:
        - name: kubeapi-proxy
          image: k8spatterns/kubeapi-proxy
        - name: config-watcher
          image: k8spatterns/curl-jq
          env:
            # The operator watches the namespace in which the operator
            # itself is installed (by using the Downward API)
            - name: WATCH_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          command:
            - 'sh'
            - '/watcher/config-watcher-operator.sh'
          volumeMounts:
            - mountPath: '/watcher'
              name: config-watcher-operator
      volumes:
        - name: config-watcher-operator
          configMap:
            name: config-watcher-operator
EOF
```

<br/>

```shell
$ kubectl logs -f $(kubectl get pods -l role=operator -o name) config-watcher
::: Starting to wait for events
::: ADDED -- config-watcher-operator
::: ADDED -- kube-root-ca.crt
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
  selector:
    app: webapp
  type: NodePort
EOF
```

<br/>

```shell
// OK
$ minikube service webapp
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# A ConfigWatch watching a configmap named "webapp-config"
# and restarts pods with label "app=webapp" in the same
# namespace.
---
kind: ConfigWatcher
apiVersion: k8spatterns.com/v1
metadata:
  name: webapp-config-watcher
spec:
  # The config map's name which should be watched
  configMap: webapp-config
  # A label selector for the pods to delete if the
  # given config map changs
  podSelector:
    app: webapp
EOF
```

<br/>

```shell
$ kubectl get configwatchers
NAME                    CONFIGMAP       PODSELECTOR
webapp-config-watcher   webapp-config   {"app":"webapp"}
```

<br/>

```shell
$ kubectl patch configmap webapp-config -p '{"data":{"message":"Greets from your smooth operator!"}}'
```

<br/>

```shell
// OK
$ minikube service webapp
```

<br/>

```shell
$ kubectl logs -f $(kubectl get pods -l role=operator -o name) config-watcher
::: Starting to wait for events
::: ADDED -- config-watcher-operator
::: ADDED -- kube-root-ca.crt
::: ADDED -- webapp-config
::: MODIFIED -- webapp-config
::::: Deleting pods with app%3Dwebapp
::::: Deleted pod webapp-75b495b7bd-gm9r6
```

<br/>

```shell
$ minikube stop && minikube delete
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
