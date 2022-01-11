---
title: "11 Working With Compute Resource Requests and Limits"
date: 2021-11-10T10:00:15Z
archives: "2021"
tags: []
author: "Kat Brookfield"
---

A long time ago, in a blog post far, far away...

...we talked about *Resource Quotas* and set *Object limits* to our new Namespace called Playpen. In this post, we will look at Compute Resource Requests and Limits.

## Compute Resource Requests and Limits
Before we proceed it is good to understand the difference between a request and a limit. As the name suggests, both are used to control resources, such as CPU and Memory, and are set on a contrainer level.

A **Request** defines how many resources is a container quaranteed to get

A **Limit** defines the maximum amount of resources a container can consume

Given these definitions, a request can be equal, but never larger than the limit, otherwise the container would never get scheduled. If you are running multiple containers in a Pod, all container values will be combined to calculate the total values.

### CPU
CPU resources are defined in millicores. If you would like a container to use half a core, you would specify 500m. It is generally advised to keep the number of CPUs under 1 to prevent issues with scheduling and provide greater flexibility. Multiple smaller replicas can be deployed to improve performance.

Another imporant thing to note about CPU is that it can be throttled. This means that if your application starts hitting the limits, Kubernetes will prevent it, even if it decreases your application perfomance.

### Memory
Memory resources are defined in bytes. You will often see it as mebibytes but you can use other values. For example, if you want your application to use 0.5GB of RAM, you would define it as 512Mi.

As Memory cannot be throttled, containers that surpass their limits will be terminated.

## Namespace Settings
While having resources defined at the container level is a good thing, it allows the creators to define whatever they want. To control this, we can set limits on the Namespace level. This may be a good idea if you are using multiple namespaces, for different environments or used by different teams.

## Resource Quotas
We have used a Resource Quota in the previous post to define our object count limits, we can use the same quota to define cpu and memory requests or limits. There are 4 settings we can use:

**requests.cpu** defines the maximum combined CPU requests in the namespace

**requests.memory** defines the maximum combined Memory requests in the namespace

**limits.cpu** defines the maximum combined CPU limits in the namespace

**limits.memory** defines the maximum combined Memory limits in the namespace

*Note: For the ease of operation I will switch my context to playpen by running kubectl config set-context - -current - -namespace=playpen*

Let's change our Resource Quota that we have previously defined. As this is set on the Namespace level, we'll list namespaces first:
```
kubectl get ns
NAME              STATUS   AGE
default           Active   176d
kube-node-lease   Active   176d
kube-public       Active   176d
kube-system       Active   176d
playpen           Active   93d
```
To see the quotas defined on our Playpen Namespace we can run the following command:
```
kubectl describe ns playpen
Name:         playpen
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
Name:     object-counts
Resource  Used  Hard
--------  ---   ---
pods      2     2
```
We can also use the following to list all existing quotas:
```
kubectl get quota
NAME            AGE   REQUEST     LIMIT
object-counts   93d   pods: 2/2
```
I am going to copy the exising quota and then modify it:
```
cp object-quota.yml resource-quota.yml
vi resource-quota.yaml
```
Add the following to the new file:
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: resource-quota
  namespace: playpen
spec:
  hard:
    requests.cpu: 500m
    requests.memory: 512Mi
    limits.cpu: 500m
    limits.memory: 512Mi
```
My nodes have 1 CPU core and 2GB of Memory, so I have decided to allocate half a core and 512MB of Memory to this namespace. 

Next I'm going to apply the new Resource quota and make sure it's assigned:
```
kubectl apply -f resource-quota.yml
resourcequota/rosource-quota created

hugo-site % kubectl describe ns playpen
Name:         playpen
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:     object-counts
 Resource  Used  Hard
 --------  ---   ---
 pods      2     2

 Name:            resource-quota
 Resource         Used  Hard
 --------         ---   ---
 limits.cpu       0     500m
 limits.memory    0     512Mi
 requests.cpu     0     500m
 requests.memory  0     512Mi

No LimitRange resource.
```
We can also use kubectl get quota to see the quotas and their usage:
```
kubectl get quota
NAME             AGE   REQUEST                                          LIMIT
object-counts    93d   pods: 2/2
resource-quota   3s    requests.cpu: 0/500m, requests.memory: 0/512Mi   limits.cpu: 0/500m, limits.memory: 0/512Mi
```
As I have my Pod quota set to 2, I will list existing pods and delete one of them:
```
kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
lily-pod    1/1     Running   0          93d
lily-pod2   1/1     Running   0          93d

kubectl delete pod lily-pod2
pod "lily-pod2" deleted
```
Now I'm going to add my requests and limits to my pod definition file *lilypod.yml*:
```
apiVersion: v1
kind: Pod
metadata:
  name: lily-pod2
  namespace: playpen
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'while true; do sleep 3600; done']
    resources:
      requests:
        memory: 256Mi
        cpu: 250m
      limits:
        memory: 512Mi 
        cpu: 250m
```
I will now recreate the pod:
```
kubectl apply -f lilypod.yml
pod/lily-pod2 created

kubectl get quota
NAME             AGE     REQUEST                                                 LIMIT
object-counts    93d     pods: 2/2
resource-quota   4m44s   requests.cpu: 250m/500m, requests.memory: 256Mi/512Mi   limits.cpu: 250m/500m, limits.memory: 512Mi/512Mi
```
To test the Quota, I am going to delete the second pod and try to recreate it with the same values:
```
kubectl delete pod lily-pod
pod "lily-pod" deleted
```
I will just change the name of the pod to lily-pod (removing2) and try to create it again:
```
kubectl apply -f lilypod.yml
Error from server (Forbidden): error when creating "lilypod.yml": pods "lily-pod" is forbidden: exceeded quota: resource-quota, requested: limits.memory=512Mi, used: limits.memory=512Mi, limited: limits.memory=512Mi
```
As you can see, this has failed because the combined value of limits has exceeded our specification.

## Limit Range
Last thing we're going to talk about in this section is called a *Limit Range*. Limit Range specifies default values that will be applied to a Container at creation, unless they are explicitely defined.
Limit Range has 4 sections:

**default** defines default limits for a container

**default.Requests** defines default requests for a container

**max** defines maximum limits that can be set for a container

**min** defines minimum limits that can be set for a container

To test this, we will create a Limit Range definition file called *limitrange.yml*:
```
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
  namespace: playpen
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 250m
    max:
      memory: 1024Mi
      cpu: 1000m
    min:
      memory: 128Mi
      cpu: 50m
    type: Container
```
Before applying it, we will delete any existing pods in the namespace. Then we will create the Limit Range:
```
kubectl apply -f limitrange.yml
limitrange/limit-range created

kubectl describe ns playpen
Name:         playpen
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:     object-counts
 Resource  Used  Hard
 --------  ---   ---
 pods      0     2

 Name:            resource-quota
 Resource         Used  Hard
 --------         ---   ---
 limits.cpu       0     500m
 limits.memory    0     512Mi
 requests.cpu     0     500m
 requests.memory  0     512Mi

Resource Limits
 Type       Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---    ---  ---------------  -------------  -----------------------
 Container  cpu       50m    1    250m             500m           -
 Container  memory    128Mi  1Gi  256Mi            512Mi          -
```
Limit range is in place and resources used are show as zero.
We will now modify our pod definition by removing the resources we've previously defined:
```
apiVersion: v1
kind: Pod
metadata:
  name: lily-pod2
  namespace: playpen
spec:
  containers:
  - name: busybox
    image: busybox
    command: ['sh', '-c', 'while true; do sleep 3600; done']
```
Deploy the pod and check the namespace again:
```
kubectl apply -f lilypod.yml
pod/lily-pod created

kubectl describe ns playpen
Name:         playpen
Labels:       <none>
Annotations:  <none>
Status:       Active

Resource Quotas
 Name:     object-counts
 Resource  Used  Hard
 --------  ---   ---
 pods      1     2

 Name:            resource-quota
 Resource         Used   Hard
 --------         ---    ---
 limits.cpu       500m   500m
 limits.memory    512Mi  512Mi
 requests.cpu     250m   500m
 requests.memory  256Mi  512Mi

Resource Limits
 Type       Resource  Min    Max  Default Request  Default Limit  Max Limit/Request Ratio
 ----       --------  ---    ---  ---------------  -------------  -----------------------
 Container  memory    128Mi  1Gi  256Mi            512Mi          -
 Container  cpu       50m    1    250m             500m           -
```
You can see the resources are now used. If we try to create another pod, we should exceed our quota:
```
kubectl apply -f lilypod.yml
Error from server (Forbidden): error when creating "lilypod.yml": pods "lily-pod2" is forbidden: exceeded quota: resource-quota, requested: limits.cpu=500m,limits.memory=512Mi, used: limits.cpu=500m,limits.memory=512Mi, limited: limits.cpu=500m,limits.memory=512Mi
```

### Cleanup
As I don't want to keep these settings in place I will delete both quotas and limitrange:
```
kubectl get limitrange
NAME          CREATED AT
limit-range   2021-11-12T13:54:51Z

kubectl delete limitrange limit-range
limitrange "limit-range" deleted

kubectl delete quota object-counts
resourcequota "object-counts" deleted

kubectl delete quota resource-quota
resourcequota "resource-quota" deleted

kubectl describe ns playpen
Name:         playpen
Labels:       <none>
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```
Last thing I'm going to do is to set my context back to default:
```
kubectl config set-context --current --namespace=default
```

This concludes today's post, see you next time!