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

We can then add Minikube executable to the path:

```shell
$ sudo mkdir -p /usr/local/bin/
$ sudo install minikube /usr/local/bin/
```

And now, we can create our Kubernetes cluster:
```shell
$ minikube start --driver=docker
```

Once the start command executes successfully, we'll see a message:
```shell
Done! kubectl is now configured to use "minikube"
For best results, install kubectl: https://kubernetes.io/docs/tasks/tools/install-kubectl/
```

Upon running the minikube status command, we should see the kubelet status as “Running“:
```shell
m01
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

Next, we need to set up the kubectl binary to be able to run the Kubernetes commands. Let's download the binary and make it executable:
```shell
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s \ 
  https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl &&
  chmod +x ./kubectl
```

Let's now move this to the path:
```shell
$ sudo mv ./kubectl /usr/local/bin/kubectl
```

We can verify the version using:
```shell
$ kubectl version
```

# Building Images Using Kaniko
Now that we have a Kubernetes cluster ready, let's start building an image using Kaniko.

First, we need to create a local directory that will be mounted in Kaniko container as the build context.

For this, we need to SSH to the Kubernetes cluster and create the directory:
```shell
$ minikube ssh
$ mkdir kaniko && cd kaniko
```
Next, let's create a Dockerfile which pulls the Ubuntu image and echoes a string “hello”:

```shell
$ echo 'FROM ubuntu' >> dockerfile
$ echo 'ENTRYPOINT ["/bin/bash", "-c", "echo hello"]' >> dockerfile
```
If we run cat dockerfile now, we should see:

```shell
FROM ubuntu
ENTRYPOINT ["/bin/bash", "-c", "echo hello"]
```

And lastly, we'll run the pwd command, to get the path to the local directory which later needs to be specified in the persistent volume.
The output for this should be similar to:

```shell
/home/docker/kaniko
```

And finally, we can abort the SSH session:

```shell
$ exit
```

# Kaniko Executor Image Arguments

Before proceeding further with creating the Kubernetes configuration files, let's take a look at some of the arguments that the Kaniko executor image requires:

- Dockerfile (–dockerfile) – File containing all the commands required to build the image
- Build context (–context) – This is similar to the build context of Docker, which refers to the directory which Kaniko uses to build the image. So far, Kaniko supports Google Cloud Storage (GCS),  Amazon S3, Azure blob storage, a Git repository, and a local directory. In this tutorial, we'll use the local directory we configured earlier.
- Destination (–destination) – This refers to the Docker registry or any similar repository to which we push the image. This argument is mandatory. If we don't want to push the image we can override the behavior by using the –no-push flag instead.


# Setting Up the Configuration Files

Let's now start creating the configuration files required for running Kaniko in the Kubernetes cluster.

First, let's create the persistent volume which provides the volume mount path that was created earlier in the cluster. Let's call the file volume.yaml:
```code
apiVersion: v1
kind: PersistentVolume
metadata:
  name: dockerfile
  labels:
    type: local
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  hostPath:
    path: /home/docker/kaniko # replace this with the output of pwd command from before, if it is different
```
