# 05 Ingress Controllers

In this section, we'll create an Nginx Ingress Controller using Helm, and show how we can dynamically configure Ingress rules to map to our services.

## What is an Ingress?

An Ingress is a collection of rules that allow inbound connections to reach the cluster services.  We can define Ingress resources as yaml, and have routing rules dynamically created to services running in the cluster.   

As an example we might want `my-app.example.org` to map to one service, and `another-app.example.org` to go to another.  

Alternatively we might want `my-app.example.org/foo` to end up at the `foo` service, but `my-app.example.org/bar` to go to `bar`

All of this (and quite a bit more!) can be configured through Ingress resources.

## Configure an Ingress Controller 

Before we can configure Ingress, we need to deploy an Ingress Controller.   An Ingress Controller is essentially a reverse proxy, which watches for updates to Kubernetes Ingress resources.  When an update occurs it is responsible for updating and reloading its configuration so the new routing rules are active.

There are a number of Ingress Controllers available, but here we'll use the stable nginx one.  

### Install with Helm

To install the Nginx Ingress Controller Helm chart, first confirm helm is running, and then run the install command.

```
> helm init
> helm upgrade --install ic nginx-ingress
 
Release "ic" has been upgraded. Happy Helming!
LAST DEPLOYED: Wed Oct 11 11:52:53 2017
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                         DATA  AGE
ic-nginx-ingress-controller  1     27s

==> v1/Service
NAME                              CLUSTER-IP  EXTERNAL-IP  PORT(S)                     AGE
ic-nginx-ingress-controller       10.0.0.145  <pending>    80:30881/TCP,443:31737/TCP  27s
ic-nginx-ingress-default-backend  10.0.0.86   <none>       80/TCP                      27s

==> v1beta1/Deployment
NAME                              DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
ic-nginx-ingress-controller       1        1        1           0          27s
ic-nginx-ingress-default-backend  1        1        1           1          27s
...
```

In the output above we can see the chart has created a _Service_ called `ic-nginx-ingress-controller` which exposes nginx on port 30881.  

Assuming your minikube instance is running on the default IP of 192.168.99.100, you should be able to hit nginx like this:

```
> curl -X GET 192.168.99.100:30881
 
default backend - 404
```

The _default backend_ response indicates that we're hitting nginx, but it has no ingress rules which matched the incoming host/path.  This backend was deployed as a _Pod_ / _Service_ combo with the Helm chart and is used as the default route when there isn't a match in the config.  It always returns a static 404 response.

### Configuring DNS

In a cloud installation of Kubernetes, creating a _Service_ of type _LoadBalancer_ would have provisioned an external cloud load balancer (e.g. AWS ELB), which we could then assign a friendly DNS entry.

In minikube we can achieve a similar result with a reverse proxy and an addition to your hosts file.  

#### Reverse Proxy

```
docker run -p 80:80 
           -e IP=192.168.99.100 
           -e PORT=30881 
           evns/local-proxy
```

This runs a local proxy, configured to listen on port `127.0.0.1` and pass all traffic through to `$DESTINATION_IP:$DESTINATION_PORT`.  

#### Hosts File

Add the following entry to the bottom of `/etc/hosts`:

```
127.0.0.1     my-host
``` 

Now we can hit the service on 

```
> curl -X GET my-host
 
default backend - 404
```

### Recap

We're now in a position to deploy a _Service_ and some _Ingress_, but first, a quick overview of what we've setup so far.

* We have `my-host` pointed at `127.0.0.1`
* We have a reverse proxy mapping `127.0.0.1` to `192.168.99.100:30881`
* We have a _Service_ for the Ingress Controller exposing port 30881.
* The Ingress Controller Service is Load Balancing across the Ingress Controller Pods
* All requests to the Ingress Controller map to the _Default Backed_ which is a simple _Pod_ / _Service_ combo.

```
           my-host
              |          
       +------v-------+
       |  127.0.0.1   |
       | local proxy  |
       +------+-------+
              |
+-------------v--------------+
|    192.168.99.100:30881    |
|             |              |
|   +---------v----------+   |
|   |      Service       |   |
|   | Ingress Controller |   |
|   +---------+----------+   |
|             |              |
|   +---------v----------+   |
|   |        Pod         |   |
|   | Ingress Controller |   |
|   +---------+----------+   |
|             |              |
|   +---------v----------+   |
|   |      Service       |   |
|   |  Default Backend   |   |
|   +---------+----------+   |
|             |              |
|   +---------v----------+   |
|   |        Pod         |   |
|   |  Default Backend   |   |
|   +--------------------+   |
|                            |
|          minikube          |
+----------------------------+
```

When you visit `http://my-host` you are routed through all of these layers to end up with a `404` from the default backend.

## Setting up Ingress

Now we have everything setup, we can deploy a simple app with some _Ingress_ rules so we can access it from outside the cluster.

Our _Ingress_ definition looks like this:

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  labels:
    app: simple-service
  annotations:
    kubernetes.io/ingress.class: "nginx"
  name: my-ingress
spec:
  rules:
  - host: my-host
    http:
      paths:
        - path: /
          backend:
            serviceName: my-service
            servicePort: 8080
```

The spec for this states that anything hitting the _Ingress Controller_  on the root path should be routed to `my-service:8080`.

Create the deployment, service, and ingress resources from this directory:

```
> kubectl apply -f .
 
deployment "my-app" configured
ingress "my-ingress" created
service "my-service" configured
```

Now when we visit `my-host` we should be routed to our app.

```
> curl -X GET my-host

Service Running (my-app-3503358887-p9grt)
```

### Deploy another app

Let's deploy a second application with its own Ingress and see what happens:

```
> kubectl apply -f ./other-app/ 

deployment "other-app" created
ingress "other-ingress" created
service "other-service" created
```

The spec for `other-ingress` looks like this:

```
spec:
  rules:
  - host: my-host
    http:
      paths:
        - path: /other-path
          backend:
            serviceName: other-service
            servicePort: 80
```

This means anything hitting `http://my-host/other-path` will be routed to other-service.

```
> curl -X GET http://my-host/other-path

<html>
<head>
	<title>Hello world!</title>
	...
```

### Overview

We've now added 2 additional service backends, with different path rules defined by Ingress.  

```
                                   my-host
                                      |
                               +------v-------+
                               |  127.0.0.1   |
                               | local proxy  |
                               +------+-------+
                                      |
+-------------------------------------+-------------------------------------+
|                            192.168.99.100:30881                           |
|                                     |                                     |
|                           +---------v----------+                          |
|                           |      Service       |                          |
|                           | Ingress Controller |                          |
|                           +---------+----------+                          |
|         anything                    |                                     |
|         unmatched         +---------v----------+      /other-path         |
|           +---------------+        Pod         +-------------+            |
|           |               | Ingress Controller |             |            |
|           |               +---------+----------+             |            |
|           |                         |                        |            |
|           |                         |  /                     |            |
|           |                         |                        |            |
| +---------v----------+    +---------v----------+   +---------v----------+ |
| |      Service       |    |      Service       |   |      Service       | |
| |  Default Backend   |    |     my-service     |   |    other-service   | |
| +---------+----------+    +---------+----------+   +---------+----------+ |
|           |                         |                        |            |
| +---------v----------+    +---------v----------+   +---------v----------+ |
| |        Pod         |    |        Pod         |   |        Pod         | |
| |  Default Backend   |    |    my-deployment   |   |  other-deployment  | |
| +--------------------+    +--------------------+   +--------------------+ |
|                                                                           |
|                                  minikube                                 |
+---------------------------------------------------------------------------+
```