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

Next, let's create a persistent volume claim for this persistent volume. We'll create a file volume-claim.yaml with:
```code
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: dockerfile-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
  storageClassName: local-storage
```

Finally, let's create the pod descriptor which comprises the executor image. This descriptor has the reference to the volume mounts specified above which in turn point to the Dockerfile we've created before.

We'll call the file pod.yaml:
```code
apiVersion: v1
kind: Pod
metadata:
  name: kaniko
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args: ["--dockerfile=/workspace/dockerfile",
            "--context=dir://workspace",
            "--no-push"] 
    volumeMounts:
      - name: dockerfile-storage
        mountPath: /workspace
  restartPolicy: Never
  volumes:
    - name: dockerfile-storage
      persistentVolumeClaim:
        claimName: dockerfile-claim
```

As mentioned before, in this tutorial we're focusing only on the image creation using Kaniko and are not publishing it. Hence we specify the no-push flag in the arguments to the executor.

With all of the required configuration files in place, let's apply each of them:
```shell
$ kubectl create -f volume.yaml
$ kubectl create -f volume-claim.yaml
$ kubectl create -f pod.yaml
```

After applying the descriptors, we can check that the Kaniko pod comes into the completed status. We can check this using "kubectl get pod -A":
```shell
NAME     READY   STATUS      RESTARTS   AGE
kaniko    0/1   Completed       0        3m
```

Now we can check the logs of this pod using kubectl logs kaniko, to check the status of the image creation and it should show the following output:
```shell
INFO[0000] Resolved base name ubuntu to ubuntu          
INFO[0000] Resolved base name ubuntu to ubuntu          
INFO[0000] Retrieving image manifest ubuntu             
INFO[0003] Retrieving image manifest ubuntu             
INFO[0006] Built cross stage deps: map[]                
INFO[0006] Retrieving image manifest ubuntu             
INFO[0008] Retrieving image manifest ubuntu             
INFO[0010] Skipping unpacking as no commands require it. 
INFO[0010] Taking snapshot of full filesystem...        
INFO[0013] Resolving paths                              
INFO[0013] ENTRYPOINT ["/bin/bash", "-c", "echo hello"] 
INFO[0013] Skipping push to container registry due to --no-push flag
```

We can see in the output that the container has executed the steps we have put in the Dockerfile.

It began by pulling the base Ubuntu image, and it ended by adding the echo command to the entry point. Since the no-push flag was specified, it didn't push the image to any repository.

Like mentioned before, we also see that a snapshot of the file system is taken before adding the entry point.

# Conclusion
In this tutorial, we've looked at a basic introduction to Kaniko. We've seen how it can be used to build an image and also setting up a Kubernetes cluster using Minikube with the required configuration for Kaniko to run.

