## Running local images in Kubernetes (Docker Desktop)

Today I would like to share with you an error that I faced when I was trying to deploy an application to Kubernetes (the standalone version included in [Docker Desktop](https://docs.docker.com/desktop/kubernetes/)) that led me to understand how Kubernetes pull images from a repository. Let's start detailing the error itself. I created the image:
``` powershell
❯ docker images
REPOSITORY           TAG           IMAGE ID          CREATED          SIZE
registry/demoapi     latest        110fb82f1b68      23 hours ago     212MB
``` 
Using the following configuration I created a deployment in Kubernetes:
``` yaml
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
Getting an error at the moment to pull the image:
```
❯ kubectl get pods
NAME                                       READY   STATUS         RESTARTS   AGE
demoapi-test-deployment-7b8f5d4fc7-8fd7t   0/1     ErrImagePull   0          14s
``` 
Then a question came to me, why is Kubernetes trying to pull the image If it's already on my machine?. Looking at the pod information, I found that Kubernetes set the image pull policy to *"Always*":
```
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
Then I went to the [Kubernetes documentation](https://kubernetes.io/docs/concepts/containers/images/) where I found the logic used to assign the default image pull policy:
> - If you omit the imagePullPolicy field, and the tag for the container image is :latest, imagePullPolicy is automatically set to Always.
> - If you omit the imagePullPolicy field, and you don't specify the tag for the container image, imagePullPolicy is automatically set to Always.
> - If you omit the imagePullPolicy field, and you specify the tag for the container image that isn't :latest, the imagePullPolicy is automatically set to IfNotPresent.

After reading that, the fix was pretty simple. I decided to specify an image pull policy:
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
```
❯ kubectl get deployments
NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
demoapi-test-deployment   1/1     1            1           2s
```
