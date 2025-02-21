---
title: "Running local images in Kubernetes with KIND and Podman"
datePublished: Fri Feb 21 2025 21:54:38 GMT+0000 (Coordinated Universal Time)
cuid: cm7fb6y25000009kz9ep0dpyb
slug: running-local-images-in-kubernetes-with-kind-and-podman
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1740153169060/bd113080-fb02-465f-8aa0-c8aee9b97a34.png
tags: kubernetes, podman, kind

---

[Docker Desktop](https://www.docker.com/products/docker-desktop/) is probably the most popular choice for using containers locally. It also includes a standalone version of Kubernetes, simplifying the process of running local clusters. However, there are alternative options available. [Podman Desktop](https://podman-desktop.io/docs/intro) along with its [KIND extension](https://podman-desktop.io/docs/kind), offers a similar capability.

[Podman](https://podman.io/) is an open-source tool for managing containers that can be used as a direct replacement for Docker. It uses the same [CLI commands](https://docs.podman.io/en/latest/Commands.html) as Docker and even offers a [`podman-compose`](https://docs.podman.io/en/v5.1.1/markdown/podman-compose.1.html) tool as an alternative to the popular [`docker-compose`](https://docs.docker.com/compose/).

[KIND (Kubernetes In Docker)](https://kind.sigs.k8s.io/) is a tool designed to run Kubernetes clusters inside containers. This means each Kubernetes node is a container, so the control plane and worker nodes run inside them.

This characteristic leads to differences when **using a local image within the Kubernetes cluster**. With Docker Desktop, the Kubernetes containers run directly on the host, and executing the `docker ps` command will display all containers related to the Kubernetes cluster. In contrast, running `podman ps` will only show the following:

```powershell
CONTAINER ID NAMES                      PORTS                                                                   
5e17653c7d7d kind-cluster-control-plane 0.0.0.0:9090->80/tcp, 0.0.0.0:9443->443/tcp, 127.0.0.1:52517->6443/tcp  
```

KIND uses [ContainerD](https://containerd.io/) as the container runtime inside the `kind-cluster-control-plane` container. This setup allows us to use [`crictl`](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md) to list the containers by running the command `podman exec -it kind-cluster-control-plane crictl ps`:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1740166690229/09685cd4-aa6b-44e3-b4a3-5576b9e506f4.png align="center")

The same issue happens with images. When we build an image locally, it's only available on our host. Therefore, we have to move the image into the `kind-cluster-control-plane` container to use it. Fortunately, KIND has a command to do just that:

```powershell
kind load --name <KIND_CLUSTER_NAME> docker-image <MY_LOCAL_IMAGE>
```

> Use `kind get clusters` to show the name of the cluster.

However, it fails with Podman because the implementation of the [`docker-image`](https://fig.io/manual/kind/load/docker-image) command is closely tied to Docker itself. So, as a workaround, we can use two other commands to achieve the same result:

```powershell
podman save -o myimage.tar <MY_LOCAL_IMAGE>
kind load --name <KIND_CLUSTER_NAME> image-archive myimage.tar
```

The [`podman save`](https://docs.podman.io/en/v5.3.0/markdown/podman-save.1.html) command allows us to save an image to a file, while the [`image-archive`](https://fig.io/manual/kind/load/image-archive) command enables loading an image from a file into the cluster. By executing the command `podman exec -it kind-cluster-control-plane crictl images` we can now see our image included in the local images of the `kind-cluster-control-plane` container.

Thank you, and happy coding.