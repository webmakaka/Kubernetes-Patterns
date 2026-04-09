# Периодическое задание (Periodic Job)

<br/>

https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/

<br/>

```bash
$ mkdir logs
$ sudo chmod -R 777 ./logs/
$ minikube --profile marley-minikube mount $(pwd)/logs:/tmp/example &
```

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
# Persistent volume mapping a hostPath. Works only on 1-node clusters like Minikube
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 20Mi
  # Storageclass is important here otherwise the PVC won't bind
  storageClassName: standard
  hostPath:
    # Mount by Minikube from local directory during 'minikube start'
    path: /tmp/example
---
# Persistent Volume Claim required by our random service
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: random-generator-log
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Mi
  volumeName: example
EOF
```

<br/>

```yaml
$ cat << 'EOF' | kubectl create -f -
apiVersion: batch/v1
kind: CronJob
metadata:
  name: random-generator
spec:
  # Every three minutes
  schedule: '*/3 * * * *'
  jobTemplate:
    spec:
      # Template for the job to generate
      template:
        spec:
          containers:
            - image: k8spatterns/random-generator:1.0
              name: random-generator
              command:
                - java
                # Use / as classpath to pick up the class file
                # Class running batch job
                - RandomRunner
                # 1. Arg: File to store data (on a PV)
                - /tmp/logs/random.log
                # 2. How many iterations
                - '10000'
              # Mount directory to where to write the results
              volumeMounts:
                - mountPath: /tmp/logs
                  name: log-volume
          restartPolicy: OnFailure
          volumes:
            - name: log-volume
              persistentVolumeClaim:
                claimName: random-generator-log
EOF
```

<br/>

По умолчанию CronJob хранит историю только последних 3 успешных и 1 неудачного завершенного задания.

<br/>

```bash
$ kubectl get jobs
NAME                        STATUS     COMPLETIONS   DURATION   AGE
random-generator-29591898   Complete   1/1           36s        8m43s
random-generator-29591901   Complete   1/1           49s        5m43s
random-generator-29591904   Complete   1/1           64s        2m43s
```

<br/>

```bash
$ kubectl delete cronjob random-generator
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
