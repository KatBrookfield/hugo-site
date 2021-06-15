---
title: "7 Configuring Topology Spread Constraints"
date: 2021-06-15T19:49:49+01:00
archives: "2021"
tags: []
author: "Kat Brookfield"
---

In previous post, we have scaled our deployment by defining number of replicas required but we weren't able to control where they get deployed. Today, we'll look into this further.

## Understanding default restart behaviour
It is important to note that if a node fails, pods running on that node are scheduled for deletion. Pods only get scheduled on a node once, when they are created, and remain running on that node until it stops or is terminated. That means that a pod is never rescheduled on another node, but a new pod is created instead.

In order to demonstrate this behaviour I have changed the number of replicas in my deployment to 2 and applied the changes:
```
katarinabrookfield@KatsMac hugo-site % vi deployment.yml
katarinabrookfield@KatsMac hugo-site % kubectl apply -f deployment.yml
deployment.apps/hugo-site-deployment configured
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-74c977df86-wmc2x   1/1     Running            0          2m44s   10.2.1.86   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-xp98x   1/1     Running            0          50m     10.2.1.85   lke27049-39949-60a53133d7b2   <none>           <none>
```

As you can see, both replicas are running on the same node. I then went into my Linode management console and powered off the node:
```
katarinabrookfield@KatsMac hugo-site % kubectl get node -o wide
NAME                          STATUS     ROLES    AGE   VERSION   INTERNAL-IP       EXTERNAL-IP       OS-IMAGE                       KERNEL-VERSION         CONTAINER-RUNTIME
lke27049-39949-60a531334464   Ready      <none>   2d    v1.20.6   192.168.195.122   139.162.192.119   Debian GNU/Linux 9 (stretch)   5.10.0-6-cloud-amd64   docker://19.3.15
lke27049-39949-60a53133d7b2   NotReady   <none>   27d   v1.20.6   192.168.142.32    176.58.114.112    Debian GNU/Linux 9 (stretch)   5.10.0-6-cloud-amd64   docker://19.3.15
lke27049-39949-60a53134727c   Ready      <none>   27d   v1.20.6   192.168.198.132   178.79.152.181    Debian GNU/Linux 9 (stretch)   5.10.0-6-cloud-amd64   docker://19.3.15
```
By using the *describe* command I could see that the status of my containers has changed to *Ready:False*:
```
katarinabrookfield@KatsMac hugo-site % kubectl describe pod hugo-site
Name:         hugo-site-deployment-74c977df86-wmc2x
.
.
Conditions:
  Type              Status
  Initialized       True
  Ready             False
  ContainersReady   True
  PodScheduled      True
  .
  .
```
Our website was no longer accessible; however, when looking at the pods, they still appeared to be running:
```
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-74c977df86-wmc2x   1/1     Running            0          7m13s   10.2.1.86   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-xp98x   1/1     Running            0          55m     10.2.1.85   lke27049-39949-60a53133d7b2   <none>           <none>
```
I have then checked my deployment and there I could see that my number of *replicas* changed to 2 unavailable. As that was 100%, it has violated the default policy which tolerates 25% unavailability. Our condition has also changed to *Available:False* because the minimum number was no longer available:
```
katarinabrookfield@KatsMac hugo-site % kubectl describe deployment hugo-site
Name:                   hugo-site-deployment
Namespace:              default
CreationTimestamp:      Tue, 15 Jun 2021 19:01:37 +0100
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               project=hugo-site
Replicas:               2 desired | 2 updated | 2 total | 0 available | 2 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  project=hugo-site
  Containers:
   hugo-pod:
    Image:        katbrookfield/hugo-site
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      False   MinimumReplicasUnavailable
OldReplicaSets:  <none>
NewReplicaSet:   hugo-site-deployment-74c977df86 (2/2 replicas created)
```
Replicasets showed 0 Ready:
```
katarinabrookfield@KatsMac hugo-site % kubectl get rs
NAME                              DESIRED   CURRENT   READY   AGE
hugo-site-deployment-74c977df86   2         2         0       65m
```
After few minutes, this has changed to 2 Ready:
```
katarinabrookfield@KatsMac hugo-site % kubectl get rs
NAME                              DESIRED   CURRENT   READY   AGE
hugo-site-deployment-74c977df86   2         2         2       68m
```
Checking the pods again, we could see the old ones were marked as *Terminating* and new ones were deployed and *Running*:
```
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-74c977df86-2sh5n   1/1     Running            0          2m5s   10.2.4.6    lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-69qtt   1/1     Running            0          2m5s   10.2.4.3    lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-wmc2x   1/1     Terminating        0          12m    10.2.1.86   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-74c977df86-xp98x   1/1     Terminating        0          60m    10.2.1.85   lke27049-39949-60a53133d7b2   <none>           <none>
```
Our deployment now showed total number of available replicas as 2 and the condition has changed to *Available:True* as the minimum number of replicas was available again:
```
katarinabrookfield@KatsMac hugo-site % kubectl describe deployment hugo-site
Name:                   hugo-site-deployment
Namespace:              default
CreationTimestamp:      Tue, 15 Jun 2021 19:01:37 +0100
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               project=hugo-site
Replicas:               2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  project=hugo-site
  Containers:
   hugo-pod:
    Image:        katbrookfield/hugo-site
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Progressing    True    NewReplicaSetAvailable
  Available      True    MinimumReplicasAvailable
```
Once I started the old host back up, the old pods were deleted:
```
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-74c977df86-2sh5n   1/1     Running            0          8m17s   10.2.4.6   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-74c977df86-69qtt   1/1     Running            0          8m17s   10.2.4.3   lke27049-39949-60a531334464   <none>           <none>
```
As soon as our pods have been successfully replaced our website became available again.
However, they are again running on the same node.

## Configuring Topology Spread Constraints
In order to introduce some level of high-availability we are going to define rules to dictate how should our pods be spread across the cluster nodes.

### Configuring Node labels
One of the prerequisites is defining Node labels that will define our *topology domains* (a fault-domain basically). These could be based on regions, zones, nodes, or other user-defined settings. In our scenario, I will define a topology based on Nodes.
```
katarinabrookfield@KatsMac hugo-site % kubectl label node lke27049-39949-60a531334464 node=node1
node/lke27049-39949-60a531334464 labeled
katarinabrookfield@KatsMac hugo-site % kubectl label node lke27049-39949-60a53133d7b2 node=node2
node/lke27049-39949-60a53133d7b2 labeled
```
You can verify that a label has been added either by running *kubectl get nodes --show-labels* or by using the *describe* command on a node. You will see the label added to existing labels:
```
katarinabrookfield@KatsMac hugo-site % kubectl describe node lke27049-39949-60a531334464
Name:               lke27049-39949-60a531334464
Roles:              <none>
Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=g6-standard-1
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=eu-west
                    kubernetes.io/arch=amd64
                    kubernetes.io/hostname=lke27049-39949-60a531334464
                    kubernetes.io/os=linux
                    lke.linode.com/pool-id=39949
                    node=node1
```
### Adding Topology Spread Constraints to a Deployment
Before we proceed with adding the rules, we will quickly verify that the pods are running on one node only:
```
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-6fb88df68-h78sp   1/1     Running            0          82s    10.2.4.8   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-6fb88df68-lj7z9   1/1     Running            0          86s    10.2.4.7   lke27049-39949-60a531334464   <none>           <none>
```
We are now going to modify our deployment to add the new rules. Edit the deployment.yml file and enter following data:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hugo-site-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      project: hugo-site
  template:
    metadata:
      labels:
        project: hugo-site
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
        image: katbrookfield/hugo-site
```
Note: As the *topologySpreadConstraints* is a *Pod* definition, we need to add it to Pod specs.

We have added following values:

  - **maxSkew** - describes the degree to which Pods may be unevenly distributed, in our case by 1

  - **topologyKey** - is the label we have added

  - **whenUnsatisfiable** - we have defined a soft rule that will try to satisfy the condition by minimizing the skew, the default option is *DoNotSchedule* which stops the pods from being scheduled

  - **labelSelector** - matches our pod labels

We will now deploy the changes, including an increase of replicas to 5:
```
katarinabrookfield@KatsMac hugo-site % kubectl apply -f deployment.yml
deployment.apps/hugo-site-deployment configured
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-6fb88df68-6cbv5   0/1     ContainerCreating   0          2s     <none>     lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-6fb88df68-h78sp   1/1     Running             0          111s   10.2.4.8   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-6fb88df68-jgtqv   0/1     ContainerCreating   0          2s     <none>     lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-6fb88df68-lj7z9   1/1     Running             0          115s   10.2.4.7   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-6fb88df68-ml7gz   0/1     ContainerCreating   0          2s     <none>     lke27049-39949-60a53133d7b2   <none>           <none>
katarinabrookfield@KatsMac hugo-site % kubectl get pod -o wide | grep hugo
hugo-site-deployment-6fb88df68-6cbv5   1/1     Running            0          13s    10.2.4.9   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-6fb88df68-h78sp   1/1     Running            0          2m2s   10.2.4.8   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-6fb88df68-jgtqv   1/1     Running            0          13s    10.2.1.6   lke27049-39949-60a53133d7b2   <none>           <none>
hugo-site-deployment-6fb88df68-lj7z9   1/1     Running            0          2m6s   10.2.4.7   lke27049-39949-60a531334464   <none>           <none>
hugo-site-deployment-6fb88df68-ml7gz   1/1     Running            0          13s    10.2.1.5   lke27049-39949-60a53133d7b2   <none>           <none>
```
3 new replicas have now been added and evenly distribuded across the two nodes we have labeled.

Our website will now remain available even when a node fails.