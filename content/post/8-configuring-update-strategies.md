---
title: "8 Configuring Update Strategies"
date: 2021-06-30T10:27:58+01:00
archives: "2021"
tags: []
author: "Kat Brookfield"
---

One thing I did not talk about yet is how I keep the website updated. This is what we will discuss in today's post.

## Current update process
I am a firm believer in KISS rule (Keep It Simple, Stupid), especially when I am working with new technologies. While I will be looking at automating this process at some stage, for now I update my blog by executing 5 simple steps:

1. Create new Hugo Post:
    ```
    hugo new post/8-configuring-update-strategies.md
    ```
2. Build new Docker image (containing the new post):
    ```
    docker image build -t katbrookfield/hugo-site .
    ```
3. Push new Docker image to my repository:
    ```
    docker image push katbrookfield/hugo-site
    ```
4. Delete the current deployment:
    ```
    kubectl delete -f deployment.yml
    ```
5. Deploy new deployment:
    ```
    kubectl apply -f deployment.yml
    ```

You may have noticed that I am not specifying any version number, this is because I want my image to be tagged as *latest* to make sure the deployment always pulls the latest version of my image, without me having to specify that:
```
katarinabrookfield@KatsMac hugo-site % docker image ls
REPOSITORY                        TAG                                                                IMAGE ID       CREATED         SIZE
katbrookfield/hugo-site           latest                                                             31eef364522d   2 weeks ago
```

This works great for my scenario, however it is not something you would want to do with production applications. That's why we will look at different update strategies.

## Configuring Rolling Updates
First thing we need to do is to introduce versioning. We will call this Version 2.0:
```
docker image build -t katbrookfield/hugo-site:2.0 .
```
We can verify the image was created with the correct tag by running *docker image ls*
```
katbrookfield/hugo-site           2.0                                                                61247866b97f   45 seconds ago   91.2MB
```

Next step is to push the new image to our repository:
```
docker image push katbrookfield/hugo-site:2.0
The push refers to repository [docker.io/katbrookfield/hugo-site]
84eca1beb165: Pushed
aa5d443685a3: Mounted from bitnami/nginx
8119d080b3a1: Mounted from bitnami/nginx
5ad7818dce34: Mounted from bitnami/nginx
6f2f8cd6c449: Mounted from bitnami/nginx
94ec56c90ba9: Mounted from bitnami/nginx
f6116e0879e2: Mounted from bitnami/nginx
d1d57010270b: Mounted from bitnami/nginx
2dd78da57c11: Mounted from bitnami/nginx
52f530832ca9: Mounted from bitnami/nginx
7958553ef831: Mounted from bitnami/nginx
32806b284726: Mounted from bitnami/nginx
2.0: digest: sha256:7b439565adb088faff617dce65c58776ed9daaa83f1221a723ef3617663ab0d4 size: 2828
```

Now that we have the image ready, we need to tell our *Deployment* to use it. 

Also, you may have noticed that the deployment gets deleted and recreated at some point during the update process. This means that until the new deployment is deployed, my website is unavailable.

To address these points, we will update our *deployment.yml* file:
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

We have added few lines to this deployment:

  - **minReadySeconds:10** - tells the deployment to wait 10 seconds before proceeding with next update

  - **maxSurge:1** and **maxUnavailable:0** define our Rolling Update strategy and mean that an extra pod will be deployed and old one removed, another one added, and old one removed, until all of the pods will run the new version, without ever dropping below the defined number of replicas. This will allow us to update our application fully with zero downtime

  - **image: katbrookfield/hugo-site:2.0** - we have updated the value to reflect our new version

We will now apply the changes:

```
kubectl apply -f deployment.yml
```

To monitor the update rollout, run the following command:
```
kubectl rollout status deployment hugo-site-deployment
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "hugo-site-deployment" successfully rolled out
```

We can also verify that our deployment is running on the new image and Update Strategies are set:
```
kubectl describe deployment hugo-site-deployment
Name:                   hugo-site-deployment
Namespace:              default
CreationTimestamp:      Wed, 16 Jun 2021 11:07:06 +0100
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               project=hugo-site
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        10
RollingUpdateStrategy:  0 max unavailable, 1 max surge
Pod Template:
  Labels:  project=hugo-site
  Containers:
   hugo-pod:
    Image:        katbrookfield/hugo-site:2.0
```

## Rolling back an Update
Our update completed successfully and website is available, but what if it wasn't? We need to be able to roll back to previous version.

To start, we can list the rollout history for our deployment:
```
kubectl rollout history deployment hugo-site-deployment
deployment.apps/hugo-site-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
The reason why we don't see any revisions is that we did not use the *- -record* flag during our deployment.

*Note-to-self: Use - -record flag during deployments!*

Luckily this does not mean that our previous version is lost. If we look at our *Replica Sets* we can see our old set still there, just not used:
```
kubectl get rs
NAME                              DESIRED   CURRENT   READY   AGE
hugo-site-deployment-6fb88df68    0         0         0       14d
hugo-site-deployment-7998bc745d   5         5         5       17m
```

We can now undo our rollout. If we used the record flag we would have know what the revision number was. Given that I have only done this once I went back to number 1:
```
kubectl rollout undo deployment hugo-site-deployment --to-revision=1
deployment.apps/hugo-site-deployment rolled back
```
If we look at replica sets now, we can see the deployment switched back to the old replica set:
```
katarinabrookfield@KatsMac hugo-site % kubectl get rs
NAME                              DESIRED   CURRENT   READY   AGE
hugo-site-deployment-6fb88df68    5         5         5       14d
hugo-site-deployment-7998bc745d   0         0         0       30m
```
And if we look into our deployment, we can see the image set back to original:
```
katarinabrookfield@KatsMac hugo-site % kubectl describe deployment hugo-site-deployment
Name:                   hugo-site-deployment
Namespace:              default
CreationTimestamp:      Wed, 16 Jun 2021 11:07:06 +0100
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 3
Selector:               project=hugo-site
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        10
RollingUpdateStrategy:  0 max unavailable, 1 max surge
Pod Template:
  Labels:  project=hugo-site
  Containers:
   hugo-pod:
    Image:        katbrookfield/hugo-site
```

## Deploying an Update (again)
Now that we have successfully demonstrated that we can roll back our deployment, we need to publish our updates again. One important thing to keep in mind is that we have used an *imperative* command to roll back. That means that our *deployment.yml* file still defines image 2.0 as the desired version.

To deploy it, we will deploy the deployment again (this time with the record flag!):
```
kubectl apply -f deployment.yml --record
deployment.apps/hugo-site-deployment configured

kubectl rollout status deployment hugo-site-deployment
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 2 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 3 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 4 out of 5 new replicas have been updated...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "hugo-site-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "hugo-site-deployment" successfully rolled out
```

If we run the history command again, we will see this deployment recorded:
```
kubectl rollout history deployment hugo-site-deployment
deployment.apps/hugo-site-deployment
REVISION  CHANGE-CAUSE
3         <none>
4         kubectl apply --filename=deployment.yml --record=true
```

And to finish this off, we will look at the deployment to make sure the image version 2.0 is used:
```
katarinabrookfield@KatsMac hugo-site % kubectl describe deployment hugo-site-deployment
Name:                   hugo-site-deployment
Namespace:              default
CreationTimestamp:      Wed, 16 Jun 2021 11:07:06 +0100
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 4
                        kubernetes.io/change-cause: kubectl apply --filename=deployment.yml --record=true
Selector:               project=hugo-site
Replicas:               5 desired | 5 updated | 5 total | 5 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        10
RollingUpdateStrategy:  0 max unavailable, 1 max surge
Pod Template:
  Labels:  project=hugo-site
  Containers:
   hugo-pod:
    Image:        katbrookfield/hugo-site:2.0
```

There are different update strategies we can take (such as blue-green or canary deployment strategies) but in order to keep these posts readable I will leave that for next time.

If you'd like to read up on these deployment types before then, I have found a very good post here: https://harness.io/blog/continuous-verification/blue-green-canary-deployment-strategies/