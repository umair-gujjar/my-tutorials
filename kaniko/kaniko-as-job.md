### Create Docker Images without Docker daemon (Kaniko)

The well-known security flaw in Docker is that it requires root access to build your Docker images with the Docker daemon.
We know that we should be careful when we are using root access. This article will help to understand the downsides of using docker and
one of the Docker alternative (Kaniko) to mitigate the security issues.

# How Kaniko works
- Reads the specified Dockerfile.
- Extracts the base image (specified in the FROM directive) into the container filesystem.
- Runs each command in the Dockerfile individually.
- Takes a snapshot of the userspace filesystem after every run.
- Appends the snapshot layer to the base layer on each run.
Because of this, Kaniko does not depend on a Docker daemon.

# Start with Kaniko:
We will use Kaniko inside a Kubernetes Cluster. To get started with Kaniko and to follow the next steps, we assume that you have the following set-up:
- A running Kubernetes cluster with permissions to create, list, update and delete jobs, services, pods, and secrets.
- A GitHub account for storing the Dockerfile and Kubernetes manifests.
- A Docker Hub account for hosting container images.

# Create Secret for Container Registry
It’s necessary to authenticate with the container registry to push the built image. So ensure that its created in the cluster.
You will need the following:
- `docker-server` — The Docker registry server where you need to host your images. If you are using Docker Hub use https://index.docker.io/v1/.
- `docker-username` — The Docker registry username.
- `docker-password` — The Docker registry password.
- `docker-email` — The email configured on the Docker registry.

Run the following command, substituting the necessary values:
```code
kubectl create secret docker-registry regcred --docker-server=<docker-server> --docker-username=<username> --docker-password=<password> --docker-email=<email>
```

For a testing I am using nignx image and I have already dockerfile and kaniko yaml job & test loads to test the image.
First lets take a look at Dockerfile.

```code
FROM nginx
RUN echo 'This image is created by kaniko' > /usr/share/nginx/html/index.html
```

The Dockerfile contains two steps. It declares the base image to nginx and writes This image is created by kanikoto /usr/share/nginx/html/index.html.
We should get that as a response when we hit the NGINX endpoint.

Let’s see what the kaniko.yaml looks like:
```code
apiVersion: batch/v1
kind: Job
metadata:
  name: kaniko
spec:
  template:
    spec:
      containers:
      - name: kaniko
        image: gcr.io/kaniko-project/executor:latest
        args: ["--dockerfile=Dockerfile",
               "--context=git://github.com/sarvabhowma1995/kaniko.git#refs/heads/main",
               "--destination=jsarvabhowma/ownnginx:v1"]
        volumeMounts:
        - name: kaniko-secret
          mountPath: "/kaniko/.docker"
      restartPolicy: Never
      volumes:
      - name: kaniko-secret
        secret:
          secretName: regcred
          items:
          - key: .dockerconfigjson
            path: config.json
```

The manifest creates a container using the gcr.io/kaniko-project/executor:latest image and runs it with the following arguments:
- docker-file — The path of the Docker file, relative to the context.
- context — The Docker context. In this case, we’ve indicated our GitHub repository
- destination — The Docker repository to push the built image.
Additionally, it also mounts a docker config JSON file on /kaniko/.docker to authenticate with the Docker repository. We defined this in the previous section.

# Build the Container Image Using Kaniko

Build the Container Image Using Kaniko
```sh
$ kubectl apply -f kaniko.yaml
```

# Testing the created Image:

To test this we have created simple deployment file to use the kaniko created image and print the page.
nginx-deployment.yaml file:
```code
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: jsarvabhowma/ownnginx:v1
        ports:
        - containerPort: 80
```

Here under spec, containers, image section i have used the custom image name which we have created through the kaniko.
nginx-service.yaml file:
```code
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: NodePort
```

In the above service we are exposing the app to the internet to test.
Now apply these two manifest files to create the application with the image which we have created through kaniko.

```sh
$ kubectl apply -f nginx-deployment.yaml
$ kubectl apply -f nginx-service.yaml
$ kubectl get all
```
