---
layout: post
title: Kubernetes with data encryption on bare metal
---

If you’ve a bunch of standalone VPS servers (perhaps from different providers), 
or your own physical servers, then you might consider running Kubernetes on them. 
Most of the posts I found dealt with creating a cluster on a single cloud provider, 
such as AWS. What I have in mind is a resilient, high-availability cluster, 
built on VPS vms from multiple providers. The 
[kubernetes documentation](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
on installing kubeadm and using it to create  a cluster is very helpful. 
There are a few points worth mentioning:

 * If you don’t plan on running demanding applications, and you'd like to 
skimp a bit, then you can try 1GB RAM for the manager, and as little as
512MB for the workers. I also had swap enabled, and that supposedly "kills
performance". If you don't want to have swap enabled, then I'd suggest
going with the safer option of 2GB for the manager and 1GB each for the nodes
at least.
 * To use swap, you will need to give special arguments to the `kubelet` to
prevent it from bailing out.

My configuration used `Ubuntu 16.04 LTS` with 1GB memory for master, and
512 MB for each node. I used 4 VPSes: 1 master, and 3 workers.

* TOC
{:toc}

## Install required software on all hosts

### Install docker

Install `docker-ce` following 
[these steps] (https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce).

### Install kubernetes components

```shell
apt-get update && apt-get install -y apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubelet kubeadm kubectl kubernetes-cni
```

### `kubelet` and swap issues
Now you will face some grief when you realize the kubelet did not start:
```
# systemctl status kubelet
# journalctl -xeu kubelet --no-pager
```

There are a couple of things to note. `kubelet` will fail to start if
swap is enabled. So, you can either disable swap or add `--fail-swap-on=false`
to the `kubelet` startup command.

```
# cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --node-ip=10.39.0.0"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode=Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Environment="KUBELET_CADVISOR_ARGS=--cadvisor-port=0"
Environment="KUBELET_CERTIFICATE_ARGS=--rotate-certificates=true --cert-dir=/var/lib/kubelet/pki"
Environment="KUBELET_EXTRA_ARGS=--fail-swap-on=false"
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_CADVISOR_ARGS $KUBELET_CERTIFICATE_ARGS $KUBELET_EXTRA_ARGS
```

Reload and restarting the kubelet:
```
# systemctl daemon-reload
# service kubelet restart
```

`kubelet` will still fail to start complaining about missing PKI 
certificates. That's expected, and will be fixed once we run `kubeadm`.

## Create the cluster using kubeadm

### `kubeadm` on master

The following master `kubeadm init` command below assumes you won’t 
be using the flannel driver for the overlay network. I recommend 
using weave as it supports encryption on the wire. If you do decide 
to use flannel, make sure you add `--pod-network-cidr=10.244.0.0/16` 
to the `kubeadm init` command below. Also the IP address you use with
`--apiserver-advertise-address` should be the IP address you want the 
worker nodes to reach you on. If your worker nodes are on private 
network, then you would use the interface address of the master node 
on that network. If your worker nodes are using the public internet 
to connect, then you should be using the public IP address of the master. 

If you plan on using swap, you may need to add `--skip-preflight-checks` 
to the `kubeadm` command.

```
# kubeadm init --apiserver-advertise-address=<your-server-IP> --kubernetes-version stable-1.8
...
Your Kubernetes master has initialized successfully!
 
To start using your cluster, you need to run (as a regular user):
 
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
 
You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
http://kubernetes.io/docs/admin/addons/
 
You can now join any number of machines by running the following on each node
as root:
 
kubeadm join...
```

Copy the `kubeadm join` line and save it in a text file. 
You will need it later in setting up the worker nodes.

### Setup `kubectl` on your workstation
```
$ curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
mv kubectl /usr/local/bin
```

### Install an overlay network
The next step is to set up an overlay network, like weave or flannel. 
I use weave as offers encryption support on the wire. This means while 
packets within a node are not encrypted, the data packets between nodes 
are encrypted. Note, we are talking about data plane encryption, which 
is what your services will use to talk to each other. Securing the 
control plane is not covered in this post.

If you decide to use weave with encryption, then you should read 
[this](https://github.com/weaveworks-experiments/weave-kube/issues/38)

The simple steps I followed are same as mentioned in the article:

```
$ kubectl create secret -n kube-system generic weave-passwd --from-file=./weave-passwd
```

Download the `weave` YAML configuration file and edit it to set `WEAVE_PASSWORD`:

```
$ curl -L -o - https://git.io/weave-kube-1.6 > weave-daemonset.yml
$ diff -u weave-daemonset.yaml weave-daemonset-password.yaml 
--- weave-daemonset.yaml    2017-11-23 22:52:26.000000000 +0530
+++ weave-daemonset-password.yaml   2017-11-23 22:55:08.000000000 +0530
@@ -108,6 +108,11 @@
                     fieldRef:
                       apiVersion: v1
                       fieldPath: spec.nodeName
+                - name: WEAVE_PASSWORD
+                  valueFrom:
+                    secretKeyRef:
+                      name: weave-passwd
+                      key: weave-passwd
               image: 'weaveworks/weave-kube:2.1.1'
               livenessProbe:
                 httpGet:
```

Now apply the modified config:

```
$ kubectl apply -f weave-daemonset-password.yaml
```

That’s it!

Of course, if you don’t care about encryption, and want to simply use weave without encryption, then skip these steps and simply do:

```
$ kubectl apply -f https://git.io/weave-kube-1.6
```

Now, check your master node shows up as READY (it might take a minute..):

```
$ kubectl get nodes
```

Below are the instructions, in case you want to use the `flannel` 
network driver instead of `weave`. Remember that the `kubeadm init` 
on the master needs `--pod-network-cidr=10.244.0.0/16` for `flannel`.

```
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created
 
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
clusterrole "flannel" created
clusterrolebinding "flannel" created
```

At this point, `kubectl get nodes` should show your master as ready. 
You should also run:
```
$ kubectl get pods --all-namespaces
```

Make sure the `kube-dns` pods are running and not stuck in `ContainerCreating`. 
That can happen if you have stale files left over from a previous install. 

### Setting up the worker nodes
Paste the `kubeadm join` line on the worker nodes and they should be able 
to join the cluster. Note, you don’t run `kubeadm init` on the worker nodes.

If your worker node has multiple IP addresses or if you would like to use a 
different IP address than the default interface address 
(for example if you’re behind NAT), then you will need to edit 
`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`, and change the line:

```
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin
```

to something like:

```
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --node-ip 54.40.255.105 --hostname-override ec2-52-40-255-105.us-east-2.compute.amazonaws.com"
```

In my case one of my workers was on EC2, and it was advertising its private 
VPC address. So, instead I used it’s public IP in line above, and its public 
hostname. The public hostname is also used by cluster nodes, so just setting 
`--node-ip` didn’t work for me; I also needed to set `--hostname-override`. 
Then restart the kubelet service.

The bootstrap token that the master printed in the `kubeadm join` during `init` 
is only valid for 24 hours. So, if you need to have a worker join later, 
just log into the master, and get the new token:

```
kubeadm token create
kubeadm token list
```

kubeadm will install the PKI certificates and restart the `kubelet` service. 
So, after running the `kubeadm` command, in a few seconds you can run 
`pidof kubelet` to check that the `kubelet` is indeed up and running.

Now view the cluster:

```
$ kubectl get nodes
NAME STATUS ROLES AGE VERSION
titan Ready 4h v1.8.3
uranus Ready master 17h v1.8.3
viking Ready 3h v1.8.3
voyager Ready 4h v1.8.3
```

## Leaving the cluster

If you need a node to leave the cluster, on that node do:
```
kubeadm reset
```

Then use kubectl to let the cluster manager know:
```
kubectl delete node <node-name>
```

The `kubeadm reset` works for both the workers and the manager, 
except in the case of the manager, you wouldn’t need to run the 
`kubectl delete node` command afterwards.

## If something goes wrong
If something gets botched and you need to remove and start from scratch, 
here is the undo recipe. You can run this on all the nodes. Once the 
slate is clean, you can install the packages and start afresh.

```
kubeadm reset
dpkg --purge kubeadm kubectl kubelet kubernetes-cni
rm -rf /etc/cni /opt/cni /etc/kubernetes
```
