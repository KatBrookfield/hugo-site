---
title: "12 Upgrading Kubernetes Cluster"
date: 2022-01-11T20:10:55Z
archives: "2022"
tags: []
author: "Kat Brookfield"
---

It has been a while since I have deployed this Kubernetes cluster, so the time has come to upgrade it. The version I initially ran was v1.20.1.

When it comes to supported versions, this is the official statement on the Kubernetes.io website:

*"The Kubernetes project maintains release branches for the most recent three minor releases (1.23, 1.22, 1.21). Kubernetes 1.19 and newer receive approximately 1 year of patch support. Kubernetes 1.18 and older received approximately 9 months of patch support."*

As I want to stay within the three minor releases, I will upgrade my cluster to the latest version available.
This blog post will have two sections. First, we'll look at how to **Upgrade Kubernetes Cluster on Linode**, and in the second section, we'll look at fully **Upgrading Kubernetes Cluster built with kubeadm**.

## Upgrading Kubernetes Cluster on Linode
If you have been following this blog from the beginning, you may have noticed I am using a Cloud solution from Linode to run my Kubernetes cluster. This makes upgrades extremely easy, as I don't have to worry about upgrading the Control Plane nodes. All of the upgrades can be achieved with just few clicks.

*Important: Before you proceed, make sure all of your Deployments are spread across multiple Nodes to avoid any outages.*

1. To start, select **Kubernetes** from the left panel, and click on the highlighted **Upgrade** button next to your cluster. (Please note: the latest currently available version on Linode is 1.22):

    ![Image](/img/Upgrade_1.png)

2. To proceed, click on **Upgrade Version**:

    ![Image](/img/Upgrade_2.png)

3. Lastly, click on **Recycle All Nodes** to finish the upgrade:

    ![Image](/img/Upgrade_3.png)

4. You will see your Nodes recycled one by one, they will go through several stages:

    ![Image](/img/Upgrade_4.png)

    ![Image](/img/Upgrade_6.png)

5. Once finished, verify that all your Nodes are **Ready** again:

    ![Image](/img/Upgrade_7.png)

You will also get an email notification once the changes have been completed. That's it!

## Upgrading Kubeadm Cluster
To demonstrate a full upgrade, I have built a new Kubernetes cluster using kubeadm. The cluster consists of one **Control Plane (Master)**, and one **Worker** node. The two VM instances are running on Google Cloud Platform (N1-Standard-2 as they need 2 vCPUs).

The upgrade will be done in two steps. First, we will upgrade the Control Plane (Master) node, and then, we'll upgrade the worker.

### Upgrading Control Plane (Master) node

1. Check the current version and status of nodes:
```
katarinabrookfield@master:~$ kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   35m   v1.22.2
worker   Ready    <none>                 24m   v1.22.2
```
2. Next, drain the Master:
```
katarinabrookfield@master:~$ kubectl drain master --ignore-daemonsets
node/master cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-l6vds, kube-system/kube-proxy-qdzt6
evicting pod kube-system/coredns-78fcd69978-pk4x8
evicting pod kube-system/calico-kube-controllers-6b9fbfff44-cpm6s
evicting pod kube-system/coredns-78fcd69978-lj8zh
pod/calico-kube-controllers-6b9fbfff44-cpm6s evicted
pod/coredns-78fcd69978-lj8zh evicted
pod/coredns-78fcd69978-pk4x8 evicted
node/master evicted
```
3. Elevate your permissions to root before proceeding with next steps:
```
katarinabrookfield@master:~$ sudo -i
root@master:~#
```
4. Update Packages list and look for the latest Kubeadm version available:
```
katarinabrookfield@master:~$ sudo -i
root@master:~# apt-get update && apt-cache madison kubeadm
Hit:1 http://security.ubuntu.com/ubuntu bionic-security InRelease
Hit:2 http://europe-west2.gce.archive.ubuntu.com/ubuntu bionic InRelease
Hit:3 http://europe-west2.gce.archive.ubuntu.com/ubuntu bionic-updates InRelease
Hit:4 http://europe-west2.gce.archive.ubuntu.com/ubuntu bionic-backports InRelease
Hit:5 https://packages.cloud.google.com/apt kubernetes-xenial InRelease
Reading package lists... Done
   kubeadm |  1.23.1-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubeadm |  1.23.0-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   kubeadm |  1.22.5-00 | http://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
   .
   .
   .
```
5. Unhold Kubeadm package:
```
root@master:~# apt-mark unhold kubeadm
Canceled hold on kubeadm.
```
6. Install latest version of Kubeadm:
```
root@master:~# apt-get install -y kubeadm=1.23.1-00
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following package was automatically installed and is no longer required:
  libnuma1
Use 'apt autoremove' to remove it.
The following packages will be upgraded:
  kubeadm
1 upgraded, 0 newly installed, 0 to remove and 5 not upgraded.
Need to get 8579 kB of archives.
After this operation, 627 kB disk space will be freed.
Get:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubeadm amd64 1.23.1-00 [8579 kB]
Fetched 8579 kB in 1s (14.8 MB/s)
(Reading database ... 66049 files and directories currently installed.)
Preparing to unpack .../kubeadm_1.23.1-00_amd64.deb ...
Unpacking kubeadm (1.23.1-00) over (1.22.2-00) ...
Setting up kubeadm (1.23.1-00) ...
```
7. Hold Kubeadm package again:
```
root@master:~# apt-mark hold kubeadm
kubeadm set on hold.
```
8. Plan the Upgrade using the following command:
```
root@master:~# kubeadm upgrade plan v1.23.1
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.22.2
[upgrade/versions] kubeadm version: v1.23.1
[upgrade/versions] Target version: v1.23.1
[upgrade/versions] Latest version in the v1.22 series: v1.23.1

Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT       TARGET
kubelet     2 x v1.22.2   v1.23.1

Upgrade to the latest version in the v1.22 series:

COMPONENT                 CURRENT   TARGET
kube-apiserver            v1.22.2   v1.23.1
kube-controller-manager   v1.22.2   v1.23.1
kube-scheduler            v1.22.2   v1.23.1
kube-proxy                v1.22.2   v1.23.1
CoreDNS                   v1.8.4    v1.8.6
etcd                      3.5.0-0   3.5.1-0

You can now apply the upgrade by executing the following command:

	kubeadm upgrade apply v1.23.1

_____________________________________________________________________


The table below shows the current state of component configs as understood by this version of kubeadm.
Configs that have a "yes" mark in the "MANUAL UPGRADE REQUIRED" column require manual config upgrade or
resetting to kubeadm defaults before a successful upgrade can be performed. The version to manually
upgrade to is denoted in the "PREFERRED VERSION" column.

API GROUP                 CURRENT VERSION   PREFERRED VERSION   MANUAL UPGRADE REQUIRED
kubeproxy.config.k8s.io   v1alpha1          v1alpha1            no
kubelet.config.k8s.io     v1beta1           v1beta1             no
_____________________________________________________________________
```
9. Apply the Upgrade (press Y when asked to proceed):
```
root@master:~# kubeadm upgrade apply v1.23.1
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0111 22:07:32.930034    5419 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
[preflight] Running pre-flight checks.
[upgrade] Running cluster health checks
[upgrade/version] You have chosen to change the cluster version to "v1.23.1"
[upgrade/versions] Cluster version: v1.22.2
[upgrade/versions] kubeadm version: v1.23.1
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
.
.
.

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.23.1". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```
9. Unhold Kubelet and Kubectl packages:
```
root@master:~# apt-mark unhold kubelet kubectl
Canceled hold on kubelet.
Canceled hold on kubectl.
```
10. Upgrade Kubelet and Kubectl packages:
```
root@master:~# apt-get install -y kubectl=1.23.1-00 kubelet=1.23.1-00
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following package was automatically installed and is no longer required:
  libnuma1
Use 'apt autoremove' to remove it.
The following packages will be upgraded:
  kubectl kubelet
2 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
Need to get 28.4 MB of archives.
After this operation, 29.2 MB disk space will be freed.
Get:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubectl amd64 1.23.1-00 [8928 kB]
Get:2 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubelet amd64 1.23.1-00 [19.5 MB]
Fetched 28.4 MB in 2s (12.2 MB/s)
(Reading database ... 66049 files and directories currently installed.)
Preparing to unpack .../kubectl_1.23.1-00_amd64.deb ...
Unpacking kubectl (1.23.1-00) over (1.22.2-00) ...
Preparing to unpack .../kubelet_1.23.1-00_amd64.deb ...
Unpacking kubelet (1.23.1-00) over (1.22.2-00) ...
Setting up kubelet (1.23.1-00) ...
Setting up kubectl (1.23.1-00) ...
```
11. Hold Kubelet and Kubectl packages again:
```
root@master:~# apt-mark hold kubelet kubectl
kubelet set on hold.
kubectl set on hold.
```
12. Reload Daemon and restart Kubelet:
```
root@master:~# systemctl daemon-reload
root@master:~# systemctl restart kubelet
root@master:~#
```
13. Logout as root. Check your nodes, you will see the Master is now upgraded, but scheduling is still disabled:
```
root@master:~# exit
logout
katarinabrookfield@master:~$ kubectl get nodes
NAME     STATUS                     ROLES                  AGE   VERSION
master   Ready,SchedulingDisabled   control-plane,master   56m   v1.23.1
worker   Ready                      <none>                 45m   v1.22.2
```
14. As the last step, uncordon the Master node:
```
katarinabrookfield@master:~$ kubectl uncordon master
node/master uncordoned
katarinabrookfield@master:~$ kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   58m   v1.23.1
worker   Ready    <none>                 47m   v1.22.2
```

### Upgrading Worker Node

1. Drain the Worker Node before logging into it:
```
katarinabrookfield@master:~$ kubectl drain worker --ignore-daemonsets
node/worker cordoned
WARNING: ignoring DaemonSet-managed Pods: kube-system/calico-node-x62gm, kube-system/kube-proxy-prmdl
evicting pod kube-system/coredns-64897985d-hktbb
evicting pod kube-system/calico-kube-controllers-6b9fbfff44-l554m
evicting pod kube-system/coredns-64897985d-gxw95
pod/calico-kube-controllers-6b9fbfff44-l554m evicted
pod/coredns-64897985d-gxw95 evicted
pod/coredns-64897985d-hktbb evicted
node/worker drained
```

2. SSH to Worker node and elevate your priviledges to root:
```
katarinabrookfield@worker:~$ sudo -i
root@worker:~#
```

3. Update Packages list:
```
root@worker:~# apt-get update
Hit:1 http://security.ubuntu.com/ubuntu bionic-security InRelease
Hit:2 http://europe-west2.gce.archive.ubuntu.com/ubuntu bionic InRelease
Hit:3 http://europe-west2.gce.archive.ubuntu.com/ubuntu bionic-updates InRelease
Hit:4 http://europe-west2.gce.archive.ubuntu.com/ubuntu bionic-backports InRelease
Get:5 https://packages.cloud.google.com/apt kubernetes-xenial InRelease [9383 B]
Fetched 9383 B in 1s (15.3 kB/s)
Reading package lists... Done
Building dependency tree
Reading state information... Done
6 packages can be upgraded. Run 'apt list --upgradable' to see them.
```

4. Unhold Kubeadm package:
```
root@worker:~# apt-mark unhold kubeadm
Canceled hold on kubeadm.
```

5. Install latest version of Kubeadm:
```
root@worker:~# apt-get install -y kubeadm=1.23.1-00
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following package was automatically installed and is no longer required:
  libnuma1
Use 'apt autoremove' to remove it.
The following packages will be upgraded:
  kubeadm
1 upgraded, 0 newly installed, 0 to remove and 5 not upgraded.
Need to get 8579 kB of archives.
After this operation, 627 kB disk space will be freed.
Get:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubeadm amd64 1.23.1-00 [8579 kB]
Fetched 8579 kB in 0s (20.3 MB/s)
(Reading database ... 66049 files and directories currently installed.)
Preparing to unpack .../kubeadm_1.23.1-00_amd64.deb ...
Unpacking kubeadm (1.23.1-00) over (1.22.2-00) ...
Setting up kubeadm (1.23.1-00) ...
```

6. Hold the Kubeadm package again:
```
root@worker:~# apt-mark hold kubeadm
kubeadm set on hold.
```

7. Upgrade the Node:
```
root@worker:~# kubeadm upgrade node
[upgrade] Reading configuration from the cluster...
[upgrade] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
W0111 22:22:21.828682   17954 utils.go:69] The recommended value for "resolvConf" in "KubeletConfiguration" is: /run/systemd/resolve/resolv.conf; the provided value is: /run/systemd/resolve/resolv.conf
[preflight] Running pre-flight checks
[preflight] Skipping prepull. Not a control plane node.
[upgrade] Skipping phase. Not a control plane node.
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[upgrade] The configuration for this node was successfully updated!
[upgrade] Now you should go ahead and upgrade the kubelet package using your package manager.
```

8. Unhold Kubelet and Kubectl packages:
```
root@worker:~# apt-mark unhold kubelet kubectl
Canceled hold on kubelet.
Canceled hold on kubectl.
```

9. Upgrade Kubelet and Kubectl packages:
```
root@worker:~# apt-get install -y kubectl=1.23.1-00 kubelet=1.23.1-00
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following package was automatically installed and is no longer required:
  libnuma1
Use 'apt autoremove' to remove it.
The following packages will be upgraded:
  kubectl kubelet
2 upgraded, 0 newly installed, 0 to remove and 3 not upgraded.
Need to get 28.4 MB of archives.
After this operation, 29.2 MB disk space will be freed.
Get:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubectl amd64 1.23.1-00 [8928 kB]
Get:2 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubelet amd64 1.23.1-00 [19.5 MB]
Fetched 28.4 MB in 2s (14.7 MB/s)
(Reading database ... 66049 files and directories currently installed.)
Preparing to unpack .../kubectl_1.23.1-00_amd64.deb ...
Unpacking kubectl (1.23.1-00) over (1.22.2-00) ...
Preparing to unpack .../kubelet_1.23.1-00_amd64.deb ...
Unpacking kubelet (1.23.1-00) over (1.22.2-00) ...
Setting up kubelet (1.23.1-00) ...
Setting up kubectl (1.23.1-00) ...
```

10. Hold Kubelet and Kubectl packages again:
```
root@worker:~# apt-mark hold kubelet kubectl
kubelet set on hold.
kubectl set on hold.
```

11. Reload Daemon and restart Kubelet:
```
root@worker:~# systemctl daemon-reload
root@worker:~# systemctl restart kubelet
root@worker:~#
```

12. Logout of Worker node:
```
root@worker:~# exit
logout
katarinabrookfield@worker:~$ exit
logout
```

13. Back on Master node, uncordon Worker node:
```
katarinabrookfield@master:~$ kubectl uncordon worker
node/worker uncordoned
```

14. Verify cluster status and versions:
```
katarinabrookfield@master:~$ kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   69m   v1.23.1
worker   Ready    <none>                 58m   v1.23.1
```
You have now fully upgraded your Kubernetes Cluster, well done!

This concludes today's post, see you next time!