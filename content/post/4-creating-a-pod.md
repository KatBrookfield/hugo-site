---
title: "4 Creating a Pod"
date: 2021-06-07T22:34:21+01:00
archives: "2021"
tags: []
author: "Kat Brookfield"
---

We are now ready to deploy our Hugo app. To start, we will create a single **Pod**.

## Create a pod
Create a file called pod.yml:
```
katarinabrookfield@KatsMac hugo-site % vi pod.yml
```
Enter the following info into the pod.yml file:
```
apiVersion: v1
kind: Pod
metadata:
  name: hugo-site-pod
  labels:
    project: hugo-site
spec:
  containers:
    - name: web-ctr
      image: katbrookfield/hugo-site
      ports:
        - containerPort: 8080
```

## Deploy the app
Run the following command to deploy the app:
```
katarinabrookfield@KatsMac hugo-site % kubectl apply -f pod.yml
pod/hugo-site-pod created
```
Verify that the pod is running:
```
katarinabrookfield@KatsMac hugo-site % kubectl get pod hugo-site-pod -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP          NODE                          NOMINATED NODE   READINESS GATES
hugo-site-pod   1/1     Running   0          43s   10.2.2.21   lke27049-39949-60a53134727c   <none>           <none>
```

We are now running Hugo on Kubernetes! Next we need to define a service to be able to connect to our website.