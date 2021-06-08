---
title: "2 Creating a Docker Container"
date: 2021-06-07T22:33:34+01:00
archives: "2021"
tags: []
author: Kat Brookfield
---

Now that we have Hugo site created locally, we need to put it in a container. You don't have to use Docker as your container runtime but it still seems to be the most popular option so I'm going to stick to it.

## Creating a Docker file
I have tried to write the Dockerfile from scratch, however it is still not working exactly how I want it to. I will keep working on it and once done, I will publish an update. In the meantime, we will use a Dockerfile that was used in this [course on kube.academy](https://kube.academy/courses/hands-on-with-kubernetes-and-containers/) by Hart Hoover.

```
FROM alpine as HUGO

ENV HUGO_VERSION="0.81.0"

RUN apk add --update wget

# Install Hugo.
RUN wget --quiet https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_Linux-64bit.tar.gz && \
    tar -xf hugo_${HUGO_VERSION}_Linux-64bit.tar.gz && \
    mv hugo /usr/local/bin/hugo && \
    rm -rf hugo_${HUGO_VERSION}_Linux-64bit.tar.gz

COPY . /hugo-site

# Use Hugo to build the static site files.
RUN hugo -v --source=/hugo-site --destination=/hugo-site/public

FROM bitnami/nginx:latest
COPY --from=HUGO /hugo-site/public/ /opt/bitnami/nginx/html/

# The container will listen on port 8080 (non-privileged) using the TCP protocol.
EXPOSE 8080
```

Note that it is copying data from your current location, so make sure you are in the *hugo-site* folder we created in the previous step.

Save the Dockerfile in the same directory, you should see these folders and files:
```
katarinabrookfield@KatsMac hugo-site % ls
Dockerfile	archetypes	config.toml	content		data		layouts		resources	static		themes
```

## Building a Docker image
Run folllowing command to create a Docker image and tag it with a name:
```
katarinabrookfield@KatsMac hugo-site % docker image build -t katbrookfield/hugo-site .
```

Please note *katbrookfield* is my Docker Hub ID, replace it with your own.

To verify that your container was created, run *docker images* command to list your images:
```
katarinabrookfield@KatsMac hugo-site % docker images
REPOSITORY                        TAG                                                                IMAGE ID       CREATED          SIZE
katbrookfield/hugo-site           latest                                                             8df5d8197c7b   2 minutes ago    89.3MB
```

## Running the Docker image
To verify that the Docker image works as expected, we will start it and try to access the website:

```
katarinabrookfield@KatsMac hugo-site % docker run -d --rm --name hugo-site -p 8080:8080 katbrookfield/hugo-site
```

*-d* tells the container to run in the background

*--rm* removes container if it exists

*--name* specifies the name of the container

*-p* publishes ports to the host

To verify that the container is working properly browse to http://localhost:8080 and you should see your website.

Please note: You may see the following warning when you run the container but it will still run:
```
katarinabrookfield@KatsMac hugo-site % docker run -d --rm --name hugo-site -p 8080:8080 katbrookfield/hugo-site
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
9004731daff674ee00048dfddfd056707e2795baee2a68b862298c56173f9eae
```

## Stop the Docker container
Now that we know the container is working as expected, we can stop it. First list running containers and then stop your container ID:
```
katarinabrookfield@KatsMac hugo-site % docker ps
CONTAINER ID   IMAGE                     COMMAND                  CREATED         STATUS         PORTS                                                 NAMES
9004731daff6   katbrookfield/hugo-site   "/opt/bitnami/script…"   7 minutes ago   Up 7 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 8443/tcp   hugo-site

katarinabrookfield@KatsMac hugo-site % docker stop 9004731daff6
9004731daff6
```

## Push Docker image to registry
We will now push the image to a Docker Hub registry by running the followind command:
```
katarinabrookfield@KatsMac hugo-site % docker image push katbrookfield/hugo-site

Using default tag: latest
The push refers to repository [docker.io/katbrookfield/hugo-site]
059a786de63e: Pushed
e5b3431038b5: Mounted from bitnami/nginx
036d035a7ddd: Mounted from bitnami/nginx
5f4995d1bfc6: Mounted from bitnami/nginx
eae9c44f5210: Mounted from bitnami/nginx
8af3324da74e: Mounted from bitnami/nginx
ca637de6423c: Mounted from bitnami/nginx
45f952449930: Mounted from bitnami/nginx
f09237220f6b: Mounted from bitnami/nginx
0045847e5b98: Mounted from bitnami/nginx
64fdb1176e87: Mounted from bitnami/nginx
1ad1d903ef1d: Layer already exists
latest: digest: sha256:92f94cc3399e0f068f1d9f0d76b3262d09328003ad425bda0376dbf55e02b0a3 size: 2826
```

Again, replace the Docker Hub ID with your own.

We now need a Kubernetes cluster to run this container on so that is what we are going to create next.