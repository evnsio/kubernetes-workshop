## 01 Create a Pod

Kubernetes abstracts the notion of a container with Pods.  Generally speaking, and for the remainder of this tutorial, they can be thought of as the same thing.

We'll use `evns/simple-service` for the following steps.  This is simple serive with some convenient endpoints for demonstrating Kubernetes features.

A simple Pod spec looks like this:

```
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: simple-service
    image: evns/simple-service
    ports:
    - containerPort: 8080
```

To deploy the Pod into the cluster we run:

```
> kubectl apply -f my-pod.yml
```

We can check the status by running:

```
> kubectl get pods

NAME             READY     STATUS    RESTARTS   AGE
my-pod           1/1       Running   0          52s
```

By default, containers running in the cluster are inaccesible from the outside. To open it up, we can forward a port from the host, to a port on the container in the cluster.

```
> kubectl port-forward my-pod 8080:8080
```

To check it's working, curl the service:

```
> curl -X GET localhost:8080

Service Running (my-pod)
```

To delete everything run:

```
> kubectl delete -f my-pod.yml
```
