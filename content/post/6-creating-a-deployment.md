---
title: "6 Creating a Deployment"
date: 2021-06-09T15:59:32+01:00
archives: "2021"
tags: []
author: "Kat Brookfield"
---

Our website is now running and we can access it, however it is running in a single pod. That means that if it fails, our website will be down. To prevent this, we need to introduce some self-healing and scaling, and that's why we'll create a **Deployment**.

## Working with Deployments
Before we create a Deployment, let's have a look what happens when we delete our single pod:
```
katarinabrookfield@KatsMac hugo-site % kubectl get pod hugo-site-pod -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP          NODE                          NOMINATED NODE   READINESS GATES
hugo-site-pod   1/1     Running   0          27h   10.2.0.15   lke27049-39949-60a531334464   <none>           <none>
katarinabrookfield@KatsMac hugo-site % kubectl delete pod hugo-site-pod
pod "hugo-site-pod" deleted
katarinabrookfield@KatsMac hugo-site % kubectl get pod hugo-site-pod -o wide
Error from server (NotFound): pods "hugo-site-pod" not found
katarinabrookfield@KatsMac hugo-site % kubectl get pod hugo-site-pod -o wide
Error from server (NotFound): pods "hugo-site-pod" not found
```
As you can see, our pod is not coming back, and neither is our website, so we need to deploy it again:

```
katarinabrookfield@KatsMac hugo-site % kubectl apply -f pod.yml
pod/hugo-site-pod created
katarinabrookfield@KatsMac hugo-site % kubectl get pod hugo-site-pod -o wide
NAME            READY   STATUS    RESTARTS   AGE   IP          NODE                          NOMINATED NODE   READINESS GATES
hugo-site-pod   1/1     Running   0          5s    10.2.0.16   lke27049-39949-60a531334464   <none>           <none>
```

### Adding self-healing
Create a file called deployment.yml:
```
katarinabrookfield@KatsMac hugo-site % vi deployment.yml
```
Enter the following info into the deployment.yml file:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hugo-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      project: hugo-site
  template:
    metadata:
      labels:
        project: hugo-site
    spec: 
      containers:
      - name: hugo-pod
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        image: katbrookfield/hugo-site
```

### Deploy the Deployment
Run the following command to deploy the deployment:
```
katarinabrookfield@KatsMac hugo-site % kubectl apply -f deployment.yml
deployment.apps/hugo-site-deployment created
katarinabrookfield@KatsMac hugo-site % kubectl get pod | grep hugo
hugo-site-deployment-74c977df86-9zvgs   1/1     Running            0          72s
hugo-site-pod                           1/1     Running            0          44m
```
As you can see, now we have two pods running; one from the previous example, and one from the deployment. We will now delete both of them and wait to see if the pod will be recreated:
```
katarinabrookfield@KatsMac hugo-site % kubectl delete pod hugo-site-deployment-74c977df86-9zvgs
pod "hugo-site-deployment-74c977df86-9zvgs" deleted
katarinabrookfield@KatsMac hugo-site % kubectl delete pod hugo-site-pod
pod "hugo-site-pod" deleted
katarinabrookfield@KatsMac hugo-site % kubectl get pod | grep hugo
hugo-site-deployment-74c977df86-z8hd2   1/1     Running            0          32s
```
New pod has been deployed automatically and our website is reachable again!

### Scale the app
Another thing we can do with deployments is to scale our app by adding or removing replicas.

As with other commands, there are *imperative* commands to scale the application but I will stick to *declarative* and update the YAML definition file. This is mainly to avoid any confusion further down the line but also seems to be the best practice.

To update the file, open the deployment.yml file again:
```
katarinabrookfield@KatsMac hugo-site % vi deployment.yml
```

Change number of replicas, I have 3 nodes so I will change this to 3 replicas for now:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hugo-site-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      project: hugo-site
  template:
    metadata:
      labels:
        project: hugo-site
    spec:
      containers:
      - name: hugo-pod
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        image: katbrookfield/hugo-site
```

If we deploy the deployment again, we will see the 2 new pods being created:
```
katarinabrookfield@KatsMac hugo-site % kubectl apply -f deployment.yml
deployment.apps/hugo-site-deployment configured
katarinabrookfield@KatsMac hugo-site % kubectl get pod | grep hugo
hugo-site-deployment-74c977df86-b8r9x   0/1     ContainerCreating   0          3s
hugo-site-deployment-74c977df86-mpfgn   0/1     ContainerCreating   0          3s
hugo-site-deployment-74c977df86-z8hd2   1/1     Running             0          59m
```

## Bonus: Where are my pods?
In the previous section, I said I wanted to have 3 replicas to have them distributed over 3 nodes of my cluster. Looking at where they run, I could see they were actually deployed on 2 nodes only:

```
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-74c977df86-8xhgl   1/1     Running            0          2m37s   10.2.0.24   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-b8kv7   1/1     Running            0          2m37s   10.2.1.32   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-cs4qd   1/1     Running            0          2m37s   10.2.0.25   lke27049-39949-60a531334464   <none>           <none>
```

So I tried to change it to 5 replicas:
```
katarinabrookfield@KatsMac hugo-site % vi deployment.yml
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-74c977df86-7nlf2   1/1     Running            0          22s     10.2.0.26   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-8xhgl   1/1     Running            0          8m32s   10.2.0.24   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-b8kv7   1/1     Running            0          8m32s   10.2.1.32   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-cs4qd   1/1     Running            0          8m32s   10.2.0.25   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-xq92b   1/1     Running            0          22s     10.2.1.33   lke27049-39949-60a53133d7b2   <none>           <none>
```

and then 10...

```
katarinabrookfield@KatsMac hugo-site % kubectl apply -f deployment.yml
deployment.apps/hugo-site-deployment configured
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-74c977df86-7nlf2   1/1     Running             0          28s     10.2.0.26   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-8xhgl   1/1     Running             0          8m38s   10.2.0.24   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-b8kv7   1/1     Running             0          8m38s   10.2.1.32   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-bkwqz   0/1     ContainerCreating   0          2s      <none>      lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-cs4qd   1/1     Running             0          8m38s   10.2.0.25   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-flhn8   0/1     ContainerCreating   0          2s      <none>      lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-flv7l   0/1     ContainerCreating   0          2s      <none>      lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-q2qdg   0/1     ContainerCreating   0          2s      <none>      lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-rz5p7   0/1     ContainerCreating   0          2s      <none>      lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-xq92b   1/1     Running             0          28s     10.2.1.33   lke27049-39949-60a53133d7b2   <none>           <none>
```

And then 60... I think you know where I'm going with this.
It still only deployed on those 2 nodes even though I have 3 nodes ready:
```
katarinabrookfield@KatsMac hugo-site % kubectl get nodes
NAME                          STATUS   ROLES    AGE   VERSION
lke27049-39949-60a531334464   Ready    <none>   21d   v1.20.6
lke27049-39949-60a53133d7b2   Ready    <none>   21d   v1.20.6
lke27049-39949-60a53134727c   Ready    <none>   21d   v1.20.6
```

So I powered off one of the hosts and the pods still stayed there but marked as completed:
```
katarinabrookfield@KatsMac hugo-site % kubectl get nodes
NAME                          STATUS     ROLES    AGE   VERSION
lke27049-39949-60a531334464   NotReady   <none>   21d   v1.20.6
lke27049-39949-60a53133d7b2   Ready      <none>   21d   v1.20.6
lke27049-39949-60a53134727c   Ready      <none>   21d   v1.20.6
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-74c977df86-m9jq8   0/1     Completed          0          63m     <none>      lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-qmpwk   1/1     Running            0          63m     10.2.1.62   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-znfcl   0/1     Completed          0          63m     <none>      lke27049-39949-60a531334464   <none>           <none>
```

Next I changed the number of replicas to 5 and it still deployed only on the second node??
```
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-74c977df86-6zpv6   1/1     Running            0          24s     10.2.1.63   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-m9jq8   0/1     Completed          0          65m     <none>      lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-qmpwk   1/1     Running            0          65m     10.2.1.62   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-rss7f   1/1     Running            0          24s     10.2.1.64   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-znfcl   0/1     Completed          0          65m     <none>      lke27049-39949-60a531334464   <none>           <none>
```

I think we'll leave it here now and look at some restart policies next.

