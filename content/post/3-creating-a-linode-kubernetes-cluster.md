---
title: "3 Creating a Linode Kubernetes Cluster"
date: 2021-06-07T22:33:48+01:00
archives: "2021"
tags: []
author: Kat Brookfield
---

In previous post, we have containerized Hugo and pushed the image to our Docker Hub registry. Now we need a Kubernetes cluster to run this container on.

There are several options, from using local *kind*, building your own Kubernetes cluster to multiple cloud offerings. I will be using [Linode](https://cloud.linode.com/) to host my Kubernetes cluster as it is a simple and cost-effective solution that allows me to run this test app, as well as to deploy load-balancers and other tools further down the line.

## Creating a Linode Kubernetes Cluster
The process is very simple, browse to https://cloud.linode.com/ and create an account. Once done, follow this simple process:

1. Click on *Kubernetes* in the left pane
2. Click on *Create Cluster*
3. Add your cluster name in *Cluster Label*
4. Select your *Region* - I am using London UK
5. Select *Kubernetes version* - I am using 1.20 on purpose
6. Add nodes - I have added three 2GB Linode nodes for $30 total

The whole process takes only few minutes to complete, you should now see your cluster.

Click on your cluster to see the details and to download your Kubeconfig file. Copy the file to your *.kube* folder and rename it as *config*:
```
katarinabrookfield@KatsMac .kube %  ls | grep config
config
```
Please note: If you have an existing config you can merge the two files. If you have multiple clusters, make sure you set your context to the Linode cluster:
```
katarinabrookfield@KatsMac .kube % kubectl config get-contexts
CURRENT   NAME           CLUSTER     AUTHINFO         NAMESPACE
          kind-kind      kind-kind   kind-kind
*         lke27049-ctx   lke27049    lke27049-admin   default
```
Verify that you can access your Kubernetes cluster by running the following command:
```
katarinabrookfield@KatsMac .kube % kubectl get nodes
NAME                          STATUS   ROLES    AGE   VERSION
lke27049-39949-60a531334464   Ready    <none>   19d   v1.20.6
lke27049-39949-60a53133d7b2   Ready    <none>   19d   v1.20.6
lke27049-39949-60a53134727c   Ready    <none>   19d   v1.20.6
```
We are now ready to deploy our app!