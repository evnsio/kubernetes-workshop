## 04 Health Checks

The health of Pods in Kubernetes is split into two 'probes':

* `livenessProbe`: Indicates whether the Container is running. If the liveness probe fails, the kubelet kills the Container, and the Container is subjected to its restart policy.

* `readinessProbe`: Indicates whether the Container is ready to service requests. If the readiness probe fails, any services selecting the pod will stop sending traffic to it.

Probes are defined as part of the Pod spec, a simple deployment might look like this:

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: evns/simple-service
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 10
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 5
          successThreshold: 1
          failureThreshold: 1
```

To deploy pods with this specification, we run:

```
> kubectl apply -f my-deployment.yml

```

Our `evns/simple-service` container has an endpoint to toggle the health of the container.  

```
> curl -X GET http://192.168.99.100:32233/toggle-health
```

_Note: your port will probably vary here_

Checking the status of the pods we'll see one of them is in a non-ready state as the readiness probe failed.  Looking at the deployment, we'll see only 2 out of the 3 pods are available.

```
> kubectl get pods

NAME                      READY     STATUS    RESTARTS   AGE
my-app-4110576992-6q7g7   1/1       Running   0          9h
my-app-4110576992-n9l2q   1/1       Running   0          9h
my-app-4110576992-wsl2f   0/1       Running   0          9h

> kubectl get deployment

NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-app    3         3         3            2           10h
```

Soon after, the liveness probe (which runs less frequently) will fail enough times for Kubernetes to restart the container.  The health will be reset, and we'll be back to 3 pods running.

```
> kubectl get pods

NAME                      READY     STATUS    RESTARTS   AGE
my-app-4110576992-6q7g7   1/1       Running   0          9h
my-app-4110576992-n9l2q   1/1       Running   0          9h
my-app-4110576992-wsl2f   1/1       Running   1          9h

> kubectl get deployment

NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-app    3         3         3            3           10h
```

### Additional details

In this example we defined `httpGet` probes with a path and port against which probe checks will be made.  The configurable fields of a check are:

* `initialDelaySeconds`:  Number of seconds after the container has started before liveness probes are initiated.
* `timeoutSeconds`: Number of seconds after which the probe times out. Defaults to 1 second.
* `periodSeconds`: How often (in seconds) to perform the probe. Default to 10 seconds.
* `successThreshold`: Minimum consecutive successes for the probe to be considered successful after having failed. Defaults to 1.
* `failureThreshold`: Minimum consecutive failures for the probe to be considered failed after having succeeded. Defaults to 3.

We've configured `httpGet` probes here, but there are two other types:

* `exec`:  Runs a command in the container, and uses the exit code to determine the health of the container.
* `tcpSocket`: Checks whether a port is open by attempting to open a socket.
