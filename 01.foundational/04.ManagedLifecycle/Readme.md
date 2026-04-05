# Управляемый жизненный цикл (Managed Lifecycle)

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
# DeploymentConfig for starting up the random-generator
apiVersion: v1
kind: Pod
metadata:
  name: random-generator
spec:
  containers:
    - image: k8spatterns/random-generator:1.0
      name: random-generator
      env:
        # Indication for the application that it should wait for the postStart file to be
        # created.
        - name: WAIT_FOR_POST_START
          value: 'true'
      lifecycle:
        # Wait 30s seconds before setting the container to be ready.
        # The sleep is just a simulation for any lengthy startup code to be run
        # Also, log to a file which then is picked up by the application
        postStart:
          exec:
            command:
              - sh
              - -c
              - sleep 30 && echo "Wake up!" > /tmp/postStart-done
        # Call out to the /shutdown endpoint. Check logs for the result
        preStop:
          httpGet:
            port: 8080
            path: shutdown
EOF
```

<br/>

```bash
$ kubectl get pods -w
NAME               READY   STATUS    RESTARTS   AGE
random-generator   1/1     Running   0          35s
```

<br/>

```bash
$ kubectl logs -f random-generator &
```

<br/>

```
23:58:18.902 [main] INFO io.k8spatterns.examples.RandomGeneratorApplication - Waiting for postStart to be finished ....
23:58:28.904 [main] INFO io.k8spatterns.examples.RandomGeneratorApplication - Waiting for postStart to be finished ....
23:58:38.905 [main] INFO io.k8spatterns.examples.RandomGeneratorApplication - Waiting for postStart to be finished ....
23:58:48.905 [main] INFO io.k8spatterns.examples.RandomGeneratorApplication - postStart Message: Wake up!
```

<br/>

```bash
$ kubectl delete pod random-generator
```

<br/>

```
2026-04-05 00:04:22.233  INFO 1 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Destroying Spring FrameworkServlet 'dispatcherServlet'
2026-04-05 00:04:22.233  INFO 1 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Destroying Spring FrameworkServlet 'dispatcherServlet'
2026-04-05 00:04:22.239  WARN 1 --- [nio-8080-exec-1] org.apache.tomcat.util.net.NioEndpoint   : The executor associated with thread pool [http-nio-8080] has not fully shutdown. Some application threads may still be running.
2026-04-05 00:04:22.241  INFO 1 --- [       Thread-0] i.k.examples.RandomGeneratorApplication  : >>>> SHUTDOWN HOOK called. Possibly because of a SIGTERM from Kubernetes
2026-04-05 00:04:22.239  WARN 1 --- [nio-8080-exec-1] org.apache.tomcat.util.net.NioEndpoint   : The executor associated with thread pool [http-nio-8080] has not fully shutdown. Some application threads may still be running.
2026-04-05 00:04:22.241  INFO 1 --- [       Thread-0] i.k.examples.RandomGeneratorApplication  : >>>> SHUTDOWN HOOK called. Possibly because of a SIGTERM from Kubernetes
```

<br/><br/>

---

<br/>

<a href="https://k8s.ru/">Предложить инженеру работу / подработку на проекте с kubernetes, microservices, machine learning, big data, golang</a>
