# 05 Intro to Helm

## Motivation

Creating resources in Kubernetes is pretty simple; create a manifest yaml file and apply it with `kubectl`.  

If we're creating a deployment, for example, our manifest might look like this:

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 1
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
```

Once we've tested our deployment locally, we might want to use the same manifest to deploy our containers to production.  In production we might want more than 1 pod running, so we could create a prod copy of this file with `replicas: 3`.

We might also deploy a service which provides a stable endpoint for our pods.  The manifest for this might look like this:

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

Suppose we change the port the container exposes to 8081, we'd now have to change our two copies of the deployment and the `targetPort` for the service.  

As applications grow increasingly complex, managing multiple sets of yaml files becomes hard.  This is where Helm can help.

## Helm

Helm is a package manager for Kubernetes. Broadly speaking, it allows us to group a collection of Kubernetes manifests, template them, and supply a set of values to populate the templates at the time of install/upgrade.  This collection of manifests, values and some additional metadata comes together to form a Helm _Chart_.

### Charts

The basic structure of a simple chart looks like this:

```
my-app/
  Chart.yaml          # A YAML file containing information about the chart
  README.md           # OPTIONAL: A human-readable README file
  values.yaml         # The default configuration values for this chart
  templates/          # OPTIONAL: A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

### Tiller

Helm is comprised of a client side component, the helm CLI, and a server side component called Tiller.  Tiller runs inside of your Kubernetes cluster, and manages releases (installations) of your charts.  Tiller is installed by running:

```
helm init
```

## Example

How could Helm help with the above scenario? We could template the variable values in the manifests to something like this:

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: {{ .Values.deployment.replicas }}
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: evns/simple-service
        ports:
        - containerPort: {{ .Values.service.port }}
```
and 

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
      targetPort: {{ .Values.service.port }}
```

And supply a values.yaml file like this:

```
deployment:
  replicas: 1

service:
  port: 8080
```

If we wanted to increase the replicas in a specific environment, we could create a second yaml file with different values and pass that in at deploy time.

### Example Chart

A slightly expanded version of this example is available [here](./helm/my-chart)

To install this chart in your cluster, you can run:

```
helm upgrade --install release-name my-chart
```

Make some changes to the manifests or values and run the same command!

## Official Charts

Many Kubernetes applications now ship as Helm Charts, which drastically simplifies their configuration.  For example, to deploy Prometheus we can simply run:

```
helm install stable/prometheus
```

Or if we want the chart version controlled in our repo:

```
helm fetch --untar stable/prometheus
```

The application is fully configurable through the charts `values.yaml` which makes the installation trivial.

To view the applications available as Helm charts, either use `helm search <thing>` from the command line or visit [kubeapps.com](https://kubeapps.com).

