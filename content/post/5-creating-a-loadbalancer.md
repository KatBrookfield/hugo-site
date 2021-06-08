---
title: "5 Creating a Loadbalancer"
date: 2021-06-07T22:34:33+01:00
archives: "2021"
tags: []
author: Kat Brookfield
---

Our pod is now running but we need to be able to connect to it. To do that, we will deploy a **Service**.

One of the reasons why we are running this on a cloud-based cluster is the ability to deploy load-balancers. I know what you are thinking; *"Why are we deploying a load-balancer for a single pod?"*. At this stage, we're doing it to prepare for next steps which will include scaling-out of our application.

## Create a Service

Create a file called cloud-lb.yml:
```
katarinabrookfield@KatsMac hugo-site % vi cloud-lb.yml
```
Enter the following info into the cloud-lb.yml file:
```
apiVersion: v1
kind: Service
metadata:
  name: hugo-site-lb
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    project: hugo-site
```

## Deploy the Service
Run the following command to deploy the service:
```
katarinabrookfield@KatsMac hugo-site % kubectl apply -f cloud-lb.yml
service/hugo-site-lb created
katarinabrookfield@KatsMac hugo-site % kubectl get svc
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)        AGE
hugo-site-lb   LoadBalancer   10.128.32.145   178.79.175.217   80:32542/TCP   4s
```

You should now be able to browser to your *External-IP* and see your website.

## Adding DNS
If you are in IT and following this, chances are that you bought a domain years ago and never used it. Well, now is your chance!

I had a test domain in AWS to I have created two A records for advecti.io and www.advecti.io for my load-balancer IP.

You should now be able to access your website using your domain name!
