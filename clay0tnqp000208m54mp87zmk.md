# Testing the NGINX Ingress Controller locally (Docker Desktop)

In this post, we will explore (briefly) the [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) in Kubernetes and see how to traffic multiple host names at the same IP address (localhost).

## What is Ingress?

> Ingress exposes HTTP and HTTPS routes from outside the cluster to services within the cluster. Traffic routing is controlled by rules defined on the Ingress resource.

![Ingress.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1669468160377/Ry4yORAUD.png align="center")

To use Ingress, you will need to install an Ingress Controller. There are many available, all of which have different features:

- [HAProxy](https://github.com/haproxytech/kubernetes-ingress)
- [NGINX](https://kubernetes.github.io/ingress-nginx/)
- [AWS ALB](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/)
- [Istio](https://istio.io/latest/docs/tasks/traffic-management/ingress/kubernetes-ingress/)
- [GLBC](https://github.com/kubernetes/ingress-gce)
- [Traefik](https://doc.traefik.io/traefik/providers/kubernetes-ingress/)

Today, we will use the NGINX Ingress Controller.

## Prerequisites

- [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/) up and running. 
- [Kubernetes](https://docs.docker.com/desktop/kubernetes/) (the standalone version included in Docker Desktop).

## Installation

One of the easiest ways to install the NGINX Ingress Controller is to use [Helm](https://helm.sh/)(if you are new to Helm, please check this [post](https://blog.raulnq.com/useful-commands-for-helm)). Run the following [commands](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-helm/):

```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
helm install my-nginx-release nginx-stable/nginx-ingress
``` 

Run `kubectl get ingressclass` and `kubectl get services` to check the installation:

```bash
NAME    CONTROLLER                     PARAMETERS   AGE
nginx   nginx.org/ingress-controller   <none>       39h
``` 

```bash
NAME                            TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
nginx-release-nginx-ingress     LoadBalancer   10.99.133.25    localhost     80:31918/TCP,443:32278/TCP   39h
```

Open `http://localhost/` in your browser:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1669470387384/1bKjD53O5.png align="center")

## Using Ingress to expose Services

Let's create a `deployment.yml` file with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api-container
          image: nginx
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
``` 

A `service.yml` file as follows:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  labels:
    app: api
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: api
``` 

And the `ingress.yml` file, to route every request to `www.api.com` to the `api-service`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: www.api.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
``` 

Run `kubectl get ingress`:

```bash
NAME                                                CLASS   HOSTS                                            ADDRESS     PORTS   AGE
api-ingress                                         nginx   www.api.com                                      localhost   80      8h
```

Everything looks fine, but if we type `http://www.api.com/` in a browser, we will not be able to reach our service. That happens because nothing tells the browser to resolve that URL to our `localhost`.  Let's change that, open the `C:\Windows\System32\drivers\etc\hosts` file and add `127.0.0.1 www.api.com` at the end:

```
192.168.1.105 host.docker.internal
192.168.1.105 gateway.docker.internal
127.0.0.1 kubernetes.docker.internal
127.0.0.1 www.api.com
``` 

Open `http://www.api.com/` in your browser:

![image.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1669472058633/eLusY7zFH.png align="center")

And that's all. Thanks, and happy coding.

