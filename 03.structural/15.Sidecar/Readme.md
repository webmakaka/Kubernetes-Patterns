# Паттерн Sidecar (Sidecar)

<br/>

### Разбор примеров из книги

<br/>

```shell
$ minikube start --mount --mount-string="$(pwd)/logs:/tmp/example"
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Example for a HTTP server with a git pulling sidecar
apiVersion: v1
kind: Pod
metadata:
  name: web-app
  labels:
    project: k8spatterns
    pattern: Sidecar
spec:
  containers:
    # Main container is a stock httpd serving from /var/www/html
    - name: app
      image: centos/httpd
      ports:
        - containerPort: 80
      volumeMounts:
        - mountPath: /var/www/html
          name: source
    # Sidecar poll every minute a given repository with git
    - name: poll
      image: bitnami/git
      env:
        - name: SOURCE_REPO
          value: https://github.com/mdn/beginner-html-site-scripted
      command: ['sh', '-c']
      args:
        - |
          git clone $(SOURCE_REPO) .
          while true; do
            sleep 60
            git pull
          done
      workingDir: /var/lib/data
      volumeMounts:
        - mountPath: /var/lib/data
          name: source
  volumes:
    # The shared directory for holding the files
    - emptyDir: {}
      name: source
---
# A service which opens a NodePort is added for your convenience
# but is not necessarily required for this example:
apiVersion: v1
kind: Service
metadata:
  labels:
    project: k8spatterns
    pattern: Sidecar
  name: web-app
spec:
  ports:
    - name: http
      port: 8080
      protocol: TCP
      targetPort: 80
  selector:
    pattern: Sidecar
  # Just for demo
  type: NodePort
EOF
```

<br/>

```shell
$ minikube ip
192.168.58.2
```

<br/>

```shell
$ port=$(kubectl get svc web-app -o jsonpath='{.spec.ports[0].nodePort}')
```

<br/>

```shell
$ echo ${port}
32571
```

<br/>

```
// OK!
http://192.168.58.2:32571
```

<br/>

```shell
$ kubectl get pod web-app -o jsonpath='{.spec.containers[*].name}'
app poll
```

<br/>

```shell
$ kubectl logs web-app -c app
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.244.0.3. Set the 'ServerName' directive globally to suppress this message
```

<br/>

```shell
$ kubectl logs web-app -c poll
Cloning into '.'...
Already up to date.
Already up to date.
Already up to date.
Already up to date.
Already up to date.
Already up to date.
```

<br/>

```shell
$ minikube stop && minikube delete
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
