### An Introduction to Kaniko


# Kaniko
Kaniko is a tool to build container images from a Dockerfile. Unlike Docker, Kaniko doesn't require the Docker daemon.

Since there's no dependency on the daemon process, this can be run in any environment where the user doesn't have root access like a Kubernetes cluster.

# Installing Minikube

We'll be using Minikube to deploy Kubernetes locally. It can be downloaded as a stand-alone binary

```shell
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 &&
  chmod +x minikube
```
