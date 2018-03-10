---
layout: post
title: Kubernetes with Vagrant and Ansible
---

Once you go through the effort of manually setting up a k8s cluster with 
`kubeadm`, you will likely want to automate the process so you can quickly 
create/destroy a cluster.

This involves two steps:

 * Creating multiple VMs using `vagrant`, with the underlying VM provider being 
   `virtualbox` or some cloud provider, such as `AWS`. 
 * Provisioning and configuring software on the VMs using Ansible
   You will discover that Ansible and Vagrant play well together.

The [code is on GitHub](https://github.com/tushar-mohan/k8s-virtualbox.git).

For my setup, I used a `Ubuntu 16.04 LTS` host with `vagrant`, `ansible` and 
`virtualbox` installed on it.

* TOC
{:toc}

## Creating VMs using Vagrant
While I used Vagrant with the virtualbox provider for this purpose, in theory 
you should be able to replace virtualbox with a provider like AWS.

{% gist 1d0943eb655e64f2bb656480dbe81829 vagrantfile %}

This a multi-machine `vagrantfile`. We set the number of slaves we want in the 
cluster, in this case two. The master machine has a `hostvar` set for its IP. 
We will use this later in the ansible playbook, provision/kubernetes.yml. You 
will notice that before we call the ansible provisioner, we install python on 
all the nodes. To avoid conflicts in port-forwarding the SSH port from the guest 
to the host, we deliberately use the node number to provide an offset. The 
slaves will be labelled `node-1`, `node-2`…

{% gist 1d0943eb655e64f2bb656480dbe81829 kubernetes.yml %}
We add `avahi-daemon` and `libnss-mdns` packages, so the machines can find each
other on the local private network using just the host name. We then removed the
need for .local in the host name, by adding an entry in `/etc/resolv.conf`.

On the `master` node we run `kube-init-master.sh` -- a shell script to run
`kubeadm init`. Sometimes, the kubelet doesn't restart properly; our shell
script will retry `kubeadm init`, until it succeeds. Here's the script:
{% gist 1d0943eb655e64f2bb656480dbe81829 kube-init-master.sh %}

On the slave nodes, we run a script -- `kube-join-slave.sh` -- that will parse
the output saved from the master's init to figure out the join token.
{% gist 1d0943eb655e64f2bb656480dbe81829 kube-join-slave.sh %}

Both the scripts are written so as to bail out in case they discover a running
cluster. This way, the scripts can be safely, idempotently, without harming a
working cluster.

Once your cluster is running, it will create a file -- `kube.config` -- in the
top-level directory. You can use it as:
```
$ KUBECONFIG=$PWD/kube.config kubectl get nodes
```

The final step to getting the cluster running, is to install an overlay; in our
case `flannel`. A helper script `flannel.sh` is automatically created by
`kube-init-master.sh`:
```
$ ./flannel.sh
```
Wait a minute or two, so `kube-dns` pods get running. Then you're good to go!
