---
title: "10 Working With Namespaces"
date: 2021-08-09T15:49:16+01:00
archives: "2021"
tags: []
author: "Kat Brookfield"
---

Today, we will look at Namespaces.

Namespaces give us an option to divide our Kubernetes cluster to multiple virtual clusters. This may be useful if you want to create different environments, for example for Prod, Dev, and Test, or if you want to assign specific permissions and policies to users. It also allows you to define resource quotas, defining how many resources can a given namespace use.

## Default Namespace
Before we start creating custom namespaces, we can have a look at namespaces that already exist in our cluster by running *kubectl get namespaces* command:
```
kubectl get namespaces
NAME              STATUS   AGE
default           Active   83d
kube-node-lease   Active   83d
kube-public       Active   83d
kube-system       Active   83d
```

- **Default** namespace, as its name suggest, is a location where all of the objects get created, unless they have another namespace specified. This is where all of our *Pods*, *Services*, and *Deployments* live. 
- **kube-node-lease** contains node lease objects.
- **kube-public** is readable by all users publicly.
- **kube-system** runs system components, such as DNS.

Note: It is worth to mention that not ALL objects are namespaced, such as *namespaces* themselves or *nodes*. To get a full list, run *kubectl api-resources*:

```
kubectl api-resources
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
...
```

To see objects running in a specific namespace, you can add *--namespace* after the commands we have been using previously.

For example, to see all pods running in the *kube-system* namespace, we can run the following:
```
kubectl get pods --namespace kube-system
NAME                                      READY   STATUS    RESTARTS   AGE
calico-kube-controllers-c6f659bdc-8bjnm   1/1     Running   0          20d
calico-node-jvbcx                         1/1     Running   1          20d
calico-node-lvrvx                         1/1     Running   0          20d
calico-node-mqhgb                         1/1     Running   0          20d
coredns-5664fbb6d6-ffbmr                  1/1     Running   0          29d
coredns-5664fbb6d6-rftmq                  1/1     Running   0          29d
csi-linode-controller-0                   4/4     Running   2          5d14h
csi-linode-node-5bn6f                     2/2     Running   1          5d14h
csi-linode-node-fpsp8                     2/2     Running   0          5d14h
csi-linode-node-tbgrw                     2/2     Running   0          5d14h
kube-proxy-8gwb6                          1/1     Running   0          20d
kube-proxy-pvfnf                          1/1     Running   0          20d
kube-proxy-t2jfv                          1/1     Running   0          20d
metrics-server-5556757555-vrrzq           1/1     Running   0          56d
```

## Creating a new Namespace
Now that we know which namespaces we already have in place, we can proceed with creating a new *Namespace*. 

Let's say I have a junior member of staff called Lily. I want to create an environment for her that she can use to play with.

First, we will create it:
```
kubectl create namespace playpen
namespace/playpen created

kubectl get namespaces
NAME              STATUS   AGE
default           Active   83d
kube-node-lease   Active   83d
kube-public       Active   83d
kube-system       Active   83d
playpen           Active   8s
```

We will now deploy a simple pod to this namespace. As we mentioned previously, there are *imperative* ways of achieving this by using the *--namespace* flag, but we will stick to *declarative* way and define our namespace by adding it to pod definition.

We will create a new file called lilypod.yml and enter the following details:
```
apiVersion: v1
kind: Pod
metadata:
  name: lily-pod
  namespace: playpen
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'while true; do sleep 3600; done']
```

Next we'll deploy it:
```
kubectl create -f lilypod.yml
pod/lily-pod created
```
If we now look for our pod, we will not be able to see it:
```
kubectl get pods |grep lily
```
This is because at the moment, we are looking into the *Default* namespace:
```
kubectl config get-contexts
CURRENT   NAME           CLUSTER     AUTHINFO         NAMESPACE
          kind-kind      kind-kind   kind-kind
*         lke27049-ctx   lke27049    lke27049-admin   default
```

To see Lily's pod, we can use the *--namespace* flag to list it:
```
kubectl get pods --namespace playpen
NAME       READY   STATUS    RESTARTS   AGE
lily-pod   1/1     Running   0          7m4s
```

Another way is to switch our context to the newly created namespace. This will be also handy if we plan on doing more work in that namespace:
```
kubectl config set-context --current --namespace playpen
Context "lke27049-ctx" modified.

kubectl get pods
NAME       READY   STATUS    RESTARTS   AGE
lily-pod   1/1     Running   0          9m11s
```

## Adding Resource Quotas
As I mentioned in the beginning, we can use *Resource Quotas* to define how many resources can a namespace use. For example, it allows us to define how much RAM or CPU can an object request or use, how much storage can be consumed, but also how many objects can be deployed in the namespace.

To demonstrate this behaviour, we will first confirm that there are no quotas set in our namespace. This is the default configuration:
```
kubectl describe namespace playpen
Name:         playpen
Labels:       <none>
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```
We will now create a *Resource Quota* called *object-quota.yml*, in which we will define that we want to only allow 2 pods to be created in this namespace:
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
spec:
  hard:
    pods: "2"
```

Next we will create this quota:
```
kubectl create -f object-quota.yml
resourcequota/object-counts created
```
When we look at our namespace again, we will see that Quotas have been added:
```
kubectl describe namespace playpen
Name:         playpen
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:     object-counts
 Resource  Used  Hard
 --------  ---   ---
 pods      1     2

No LimitRange resource.
```

If we now try to deploy another pod (using the same lilypod.yml, just changing the name), we should be able to:
```
kubectl create -f lilypod.yml
pod/lily-pod2 created
```

We will see that we have already reached the limit in our namespace description:
```
kubectl describe namespace playpen
Name:         playpen
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:     object-counts
 Resource  Used  Hard
 --------  ---   ---
 pods      2     2

No LimitRange resource.
```
Let's see what happens if we try to deploy another pod:
```
kubectl create -f lilypod.yml
Error from server (Forbidden): error when creating "lilypod.yml": pods "lily-pod3" is forbidden: exceeded quota: object-counts, requested: pods=1, used: pods=2, limited: pods=2
```
The third pod failed to create as it would violate our defined quota. We will still be able to see our two previously deployed pods running successfully:
```
kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
lily-pod    1/1     Running   0          23m
lily-pod2   1/1     Running   0          28s
```

We have now demonstrated how object quotas work. 

Defining resource consumption quotas requires a bit more configuration so we will look into that in the next post.
