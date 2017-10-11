# 02 Create a Deployment

Deployments are used to manage pods. They consist of:

* A Pod specification (image, ports, labels etc.)
* A number of replicas

A simple Deployment spec looks like this:

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
      - name: simple-service
        image: evns/simple-service
        ports:
        - containerPort: 8080
```

To deploy the Pod into the cluster we run:

```
> kubectl apply -f my-deployment.yml
```

We can see the status of the deployment by running:

```
> kubectl get deployment

NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
my-app    3         3         3            3           1m
```

And the status of the Pods with:

```
> kubectl get pods

NAME                      READY     STATUS    RESTARTS   AGE
my-app-3509139055-k9gfk   1/1       Running   0          4m
my-app-3509139055-l2vvk   1/1       Running   0          4m
my-app-3509139055-wr5n4   1/1       Running   0          4m
```

We now have a deployment which has created three pods in the cluster.

## Scheduling

The deployment object doesn't just deploy the requested number of replicas once, but instead monitors continually to ensure the desired number of pods are present.   We can see this by manually deleting one:

```
> kubectl delete pod my-app-3509139055-wr5n4
> kubectl get pods

NAME                      READY     STATUS        RESTARTS   AGE
my-app-3509139055-k9gfk   1/1       Running       0          5m
my-app-3509139055-l2vvk   1/1       Terminating   0          5m
my-app-3509139055-vsksc   1/1       Running       0          4s
my-app-3509139055-wr5n4   1/1       Running       0          5m
```

Immediately after deleting one, we see a new Pod created in its place.
