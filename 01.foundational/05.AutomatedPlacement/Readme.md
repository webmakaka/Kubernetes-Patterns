# Автоматическое размещение (Automated Placement)

<br/>

### Node Selector

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
# Node selector example which only picks the node
apiVersion: v1
kind: Pod
metadata:
  name: node-selector
spec:
  containers:
    - image: k8spatterns/random-generator:1.0
      name: random-generator
  nodeSelector:
    # Simple match on labels
    disktype: ssd
EOF
```

<br/>

```bash
$ kubectl describe pod node-selector
***
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  6s    default-scheduler  0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector. no new claims to deallocate, preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

<br/>

```bash
$ kubectl get nodes
NAME              STATUS   ROLES           AGE     VERSION
marley-minikube   Ready    control-plane   2d20h   v1.35.1
```

<br/>

```bash
$ kubectl label node marley-minikube disktype=ssd
```

<br/>

```bash
$ kubectl get pods
NAME            READY   STATUS    RESTARTS   AGE
node-selector   1/1     Running   0          107s
```

<br/>

```bash
$ kubectl delete pod node-selector
```

<br/>

### Node Affinity

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
# Pod with node affinity 2
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity
spec:
  affinity:
    nodeAffinity:
      # Required rule to be fullfilled by a node for
      # this pod to be scheduled. Here we are requiring
      # a label "numberCores" with a value larger than 3
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: numberCores
                operator: Gt
                values: ['3']
      # Preferred rules which influence scheduling decision.
      # Here we don't like to be scheduled on the Minikube control-plane node
      # (but of course will be ignored if we are testing on Minikube with
      # only one node available)
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 10
          preference:
            matchFields:
              - key: metadata.name
                operator: NotIn
                values: ['minikube']
  containers:
    - image: k8spatterns/random-generator:1.0
      name: random-generator
EOF
```

<br/>

```bash
$ kubectl label node marley-minikube numberCores=4
```

<br/>

```bash
$ kubectl delete pod node-affinity
```

<br/>

### Pod Affinity

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
# Pod with Pod affinity
apiVersion: v1
kind: Pod
metadata:
  name: pod-affinity
spec:
  affinity:
    podAffinity:
      # We required to find pods with a label 'confidential=high'.
      # Then k8s checks whether the nodes on which these Pods are
      # running have a label "security-zone".
      # If so, then all nodes are looked up which have the same value of this
      # label. These nodes are then considered for being the
      # host of this Pod.
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              confidential: high
          topologyKey: security-zone
    podAntiAffinity:
      # We don't want to run on any node where a pod with the
      # label 'confidential=none' is running, but if these are
      # the only available then its still ok to be scheduled
      # on such a node
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                confidential: none
            topologyKey: kubernetes.io/hostname
  containers:
    - image: k8spatterns/random-generator:1.0
      name: random-generator
---
# Placeholder Pod for attracting the random-generator Pod
apiVersion: v1
kind: Pod
metadata:
  name: confidential-high
  labels:
    confidential: high
spec:
  containers:
    - image: nginx
      name: nginx
EOF
```

<br/>

```bash
$ kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
confidential-high   1/1     Running   0          73s
pod-affinity        0/1     Pending   0          73s
```

<br/>

```bash
$ kubectl label --overwrite node marley-minikube security-zone=high
```

<br/>

```bash
$ kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
confidential-high   1/1     Running   0          2m19s
pod-affinity        1/1     Running   0          2m19s
```

<br/>

```bash
$ kubectl delete pod confidential-high
$ kubectl delete pod pod-affinity
```

<br/>

### Taints and Tolerations

<br/>

```bash
$ kubectl taint nodes marley-minikube node-role.kubernetes.io/master="":NoSchedule
```

<br/>

Опять запускаю pod-affinity

<br/>

```bash
$ kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
confidential-high   0/1     Pending   0          39s
pod-affinity        0/1     Pending   0          39s
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
# Tolerations
apiVersion: v1
kind: Pod
metadata:
  name: tolerations
spec:
  containers:
    - image: k8spatterns/random-generator:1.0
      name: random-generator
  tolerations:
    - key: node-role.kubernetes.io/master
      operator: Exists
      effect: NoSchedule
EOF
```

<br/>

```bash
$ kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
confidential-high   0/1     Pending   0          109s
pod-affinity        0/1     Pending   0          109s
tolerations         1/1     Running   0          16s
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
