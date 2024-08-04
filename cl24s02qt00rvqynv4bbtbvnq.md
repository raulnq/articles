---
title: "Running local images in Kubernetes (Docker Desktop)"
datePublished: Mon Apr 18 2022 13:51:48 GMT+0000 (Coordinated Universal Time)
cuid: cl24s02qt00rvqynv4bbtbvnq
slug: running-local-images-in-kubernetes-docker-desktop
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1651962522968/y1pnSXlWi.png
tags: docker, kubernetes

---

Today, we want to share an error we encountered while deploying an application to Kubernetes (the standalone version included in [Docker Desktop](https://docs.docker.com/desktop/kubernetes/)). This experience helped us understand how Kubernetes pulls images from a repository. Let's start by detailing the error itself. We created the image:

```powershell
❯ docker images
REPOSITORY           TAG           IMAGE ID          CREATED          SIZE
registry/demoapi     latest        110fb82f1b68      23 hours ago     212MB
```

Using the following configuration, we created a deployment in Kubernetes:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapi-test-deployment
  labels:
    app: demoapi-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoapi-test
  template:
    metadata:
      labels:
        app: demoapi-test
    spec:
      containers:
      - name: demoapi-test-container
        image: registry/demoapi:latest
        ports:
        - containerPort: 80
```

We encountered an error while trying to pull the image:

```csharp
❯ kubectl get pods
NAME                                       READY   STATUS         RESTARTS   AGE
demoapi-test-deployment-7b8f5d4fc7-8fd7t   0/1     ErrImagePull   0          14s
```

Then a question came to us: why is Kubernetes trying to pull the image if it's already on our machine? Looking at the pod information, we found that Kubernetes set the image pull policy to `Always`:

```csharp
❯ kubectl get pod demoapi-test-deployment-7b8f5d4fc7-8fd7t -o yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2022-04-17T15:51:47Z"
  generateName: demoapi-test-deployment-7b8f5d4fc7-
  labels:
    app: demoapi-test
    pod-template-hash: 7b8f5d4fc7
  name: demoapi-test-deployment-7b8f5d4fc7-8fd7t
  namespace: default
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: demoapi-test-deployment-7b8f5d4fc7
    uid: 7a4cdd35-8f56-4e5e-ab11-efc1e8994423
  resourceVersion: "70292"
  uid: 4c430c05-f007-4a81-ad6a-396231675567
spec:
  containers:
  - image: registry/demoapi:latest
    imagePullPolicy: Always
    name: demoapi-test-container
    ports:
    - containerPort: 80
      protocol: TCP
```

Checking the [Kubernetes documentation](https://kubernetes.io/docs/concepts/containers/images/), we found the logic used to assign the default image pull policy:

> * If you omit the imagePullPolicy field, and the tag for the container image is :latest, imagePullPolicy is automatically set to Always.
>     
> * If you omit the imagePullPolicy field, and you don't specify the tag for the container image, imagePullPolicy is automatically set to Always.
>     
> * If you omit the imagePullPolicy field, and you specify the tag for the container image that isn't :latest, the imagePullPolicy is automatically set to IfNotPresent.
>     

After reading that, the fix was straightforward. We decided to specify an image pull policy:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demoapi-test-deployment
  labels:
    app: demoapi-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demoapi-test
  template:
    metadata:
      labels:
        app: demoapi-test
    spec:
      containers:
      - name: demoapi-test-container
        image: registry/demoapi:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

And problem fixed:

```csharp
❯ kubectl get deployments
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
demoapi-test-deployment   1/1     1            1           2s
```

Thank you, and happy coding.