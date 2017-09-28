## 03 Create a Service

Services are used to:

* Load-balance across Pods
* Provide a stable way to access Pods by IP or DNS
* Abstract away the volatility of Pod scheduling

The spec for a service contains:

* A selector to define the backend pods
* A set of port specifications

A simple Service spec looks like this:

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      name: web
      port: 8080
      targetPort: 8080
```

This service will load balance across any Pods with the `app: my-app` label defined.  

We can apply it in the cluster using:

```
> kubectl apply -f my-service.yml
```

We can make requests against this service and benefit from load balancing, stable DNS name, irrespective of Pod scheduling behind the scenes.

For example:

```
# Run a Pod in interactive mode:
> kubectl run my-curl -ti --rm --image tutum/curl bash

root@my-curl-1857406192-d8dj6:/# curl -X GET http://my-service:8080

Service Running (my-app-3509139055-wr5n4)
```

### Outside the Cluster

The service definition above is only accessible from within the cluster.  To expose the service externally we can change the type to `NodePort`, at which point the service will be assigned a random port on _every_ host in the cluster.

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      name: web
      port: 8080
      targetPort: 8080
  type: NodePort
```

```
> kubectl apply -f my-service.yml

> kubectl get service my-service

NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
my-service   10.0.0.59    <nodes>       8080:32233/TCP   4m
```
This service now has two ports assigned to it: `8080` for cluster internal applications, and a nodeport on `32233`.  

To test this, we can run:

```
> curl -X GET http://192.168.99.100:32233

Service Running (my-app-3509139055-vsksc)

```
