---
title: "9 Blue/Green and Canary Deployments"
date: 2021-07-25T19:35:49+01:00
archives: "2021"
tags: []
author: "Kat Brookfield"
---

Previously, we talked about Rolling Updates and how we are able to update the application with zero downtime. 

In that scenario, all old pods were gradually replaced by new pods running the updated version of our application. While this method was very efficient, it did not really allow us to perform any kind of testing of the updated application in the new setting, and in case of a failure, our only option was to fully roll back.

In order to introduce more control and testing ability into these deployments, we will now look at two additional deployment methods.

## Blue/Green Deployments
In a Blue/Green Deployment scenario, we will deploy two near identical deployments. **Blue Deployment** will be the current version of our application, and **Green Deployment** will run an updated version of our application. We will use our *Service* to switch between the two. 

First we will verify that our deployment is still there:
```
kubectl get deployments | grep hugo
hugo-site-deployment     5/5     5            5           25d
```
Next we need to update our **Docker Image**, as we did in Previous Post [http://advecti.io/2021/8-configuring-update-strategies/] 
```
docker image build -t katbrookfield/hugo-site:2.1 .
docker image push katbrookfield/hugo-site:2.1
```

We will now copy *deployment.yml* and call it *deployment-green.yml*. We will also change the name of the deployment as well as project to **hugo-site-green**. Lastly, we will change the image version to our latest release:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hugo-site-deployment-green
spec:
  replicas: 5
  selector:
    matchLabels:
      project: hugo-site-green
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        project: hugo-site-green
    spec: 
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: node
        whenUnsatisfiable: ScheduleAnyway
        labelSelector: 
          matchLabels:
            project: hugo-site-green
      containers:
      - name: hugo-pod
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        image: katbrookfield/hugo-site:2.1
```
We will now create the new deployment:
```
kubectl create -f deployment-green.yml
deployment.apps/hugo-site-deployment-green created
```
Next we will verify both deployments are available:
```
kubectl get deployments | grep hugo
hugo-site-deployment         5/5     5            5           25d
hugo-site-deployment-green   5/5     5            5           58s
```
As this stage, we could just switch our *Service* to the new green deployment. However, we can perform an additional step to make sure the application is fully tested before we route live users to it.

### Deploying a Test Green LB Service
As we have previously discussed, we need to expose our application in order to access it. We use a *Service* to do this, and in my case, a cloud load balancer.

All we will do is simply copy *cloud-lb.yml* to *cloud-lb-green.yml* and change name and project to *hugo-site-green*
```
apiVersion: v1
kind: Service
metadata:
  name: hugo-site-lb-green
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    project: hugo-site-green
```
We will now create the new service and make sure it is running.
```
kubectl create -f cloud-lb-green.yml
service/hugo-site-lb-green created

kubectl get service |grep hugo
hugo-site-lb         LoadBalancer   10.128.136.95    185.3.93.98   80:31940/TCP   47d
hugo-site-lb-green   LoadBalancer   10.128.43.224    185.3.92.37   80:30016/TCP   10s
```
When we'll browse to the External IP of the new Service, we will see our updated application. This way, we can perform testing and make sure everything is running as expected, before making the switch to Production.

It is worth noting that due to the nature of this deployment, it requires twice as many resources to run. It would be advisable to decrease the number of replicas, or completely remove the old deployment, once the switch has been made.

## Canary Deployments
Another scenario is called Canary Deployments, which allows you to roll out updates to a subset of users, and therefore test your application with live traffic.

To distinguish between the different releases we will use multiple *Labels*.
First, we will add a *track label* to *deployment.yml* under pod definition:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hugo-site-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      project: hugo-site
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        project: hugo-site
        track: stable
    spec: 
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: node
        whenUnsatisfiable: ScheduleAnyway
        labelSelector: 
          matchLabels:
            project: hugo-site
      containers:
      - name: hugo-pod
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        image: katbrookfield/hugo-site:2.0
```
Next we will copy *deployment.yml* to *deployment-canary,yml* and change the name, track label to canary, and image version. We will also change the number of replicas to 1, as we only want to deploy one pod with the new version:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hugo-site-deployment-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      project: hugo-site
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        project: hugo-site
        track: canary
    spec: 
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: node
        whenUnsatisfiable: ScheduleAnyway
        labelSelector: 
          matchLabels:
            project: hugo-site
      containers:
      - name: hugo-pod
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        image: katbrookfield/hugo-site:2.1
```
As we are not changing the project name, both deployments will be catched by the load-balancer.
A quick way of verifying the new pod has been added is by looking at *Endpoints* of our Load-balancer:
```
kubectl get endpoints |grep hugo
hugo-site-lb    10.2.1.13:8080,10.2.1.14:8080,10.2.1.15:8080 + 2 more...   47d
```
As soon as we deploy *deployment-canary.yml* with 1 replica, the number of endpoints will increase by 1:
```
kubectl create -f deployment-canary.yml
deployment.apps/hugo-site-deployment-canary created

kubectl get endpoints |grep hugo
hugo-site-lb    10.2.1.13:8080,10.2.1.14:8080,10.2.1.15:8080 + 3 more...   47d
```

This will allow us to control the amount of users affected by the change.

Each of these update strategies has advantages and disadvantages, and it is up to you to choose the best fit for your application.

For the purpose of this blog I will continue to use Rolling Updates as I always want to replace all pods with the latest release.