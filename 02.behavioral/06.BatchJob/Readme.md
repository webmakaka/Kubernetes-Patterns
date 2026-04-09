# Пакетное задание (Batch Job)

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
# A Job resource
---
apiVersion: batch/v1
kind: Job
metadata:
  # Use a generated name so that this descriptor can be
  # used multiple times with "kubectl create" without conflicts
  # because of jobs having the same names
  generateName: random-generator-
  labels:
    app: random-generator
spec:
  # Job should run 5 Pods
  completions: 5
  # 3 Pods should run in parallel
  parallelism: 3
  # Remove pods after 5 minutes when they are done
  ttlSecondsAfterFinished: 300
  template:
    metadata:
      name: random-generator
    spec:
      containers:
        - image: k8spatterns/random-generator:1.0
          name: random-generator
          command:
            - java
            # Class running batch job
            - RandomRunner
            # 1. Arg: File to store data (on a PV)
            - /tmp/logs/random.log
            # 2. How many iterations
            - '10000'
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

```bash
$ kubectl get jobs
NAME                     STATUS     COMPLETIONS   DURATION   AGE
random-generator-d7q42   Complete   5/5           22s        2m15s
```

<br/>

```bash
$ kubectl delete jobs -l app=random-generator
```

<br/>

### Indexed Job

<br/>

```yaml
$ cat << 'EOF' | kubectl create -f -
# An indexed job that splits up a file
# Run this job after job.yaml
---
apiVersion: batch/v1
kind: Job
metadata:
  # Use a generated name so that this descriptor can be
  # used multiple times with "kubectl create" without conflicts
  # because of jobs having the same names
  generateName: file-split-
  labels:
    app: random-generator
spec:
  # Completion mode needs to be set to "Indexed"
  completionMode: Indexed
  # Job should run 5 Pods, all in parallel
  completions: 5
  parallelism: 5
  template:
    metadata:
      name: file-split
    spec:
      containers:
        - image: alpine
          name: split
          # Split up based on the JOB_COMPLETION_INDEX which is set for this
          # particular job. Note that this scripts assumes that there
          # are 50000 entries in /tmp/logs/random.log
          command:
            - 'sh'
            - '-c'
            - |
              start=$(expr \$JOB_COMPLETION_INDEX \* 10000)
              end=$(expr \$JOB_COMPLETION_INDEX \* 10000 + 10000)
              awk 'NR>=\$start && NR<\$end' /tmp/logs/random.log \
                  > /tmp/logs/random-\$JOB_COMPLETION_INDEX.txt
          volumeMounts:
            - mountPath: /tmp/logs
              name: log-volume
      # Retry again if failed (this field is mandatory)
      restartPolicy: Never
      volumes:
        - name: log-volume
          persistentVolumeClaim:
            # Same volume claim that is referenced in job.yaml
            claimName: random-generator-log
EOF
```

<br/>

```bash
$ kubectl get jobs
NAME               STATUS     COMPLETIONS   DURATION   AGE
file-split-hd2tc   Complete   5/5           9s         42s
```

<br/>

```bash
$ kubectl delete jobs -l app=random-generator
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
