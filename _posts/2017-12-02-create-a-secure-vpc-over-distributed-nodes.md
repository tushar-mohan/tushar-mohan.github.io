---
layout: post
title: Create a secure VPC across cloud providers
---

If you like high availability and resilience then you probably want to create
a virtual private cloud (VPC) using commodity VPS servers and perhaps your 
own home machines. It seems a lot cheaper and safer than putting all your
eggs in the AWS or GCP or Azure basket.

The key to making it happen is connecting the nodes using a secure overlay 
network. Ideally, the overlay network should be able to work across NAT, so 
it can cross into your home network. Recently, I discovered that it’s fairly 
easy to accomplish this using [Weave Net](https://www.weave.works/).

While Weave Net attempts to solve a more challenging problem — that of 
securely, connecting containers across distributed hosts — our interest is 
primarily in making the hosts connected using Weave’s overlay network.

My setup used 4 hosts on 3 different hosting providers — AWS, Linode and Vultr. 
The last node was on my home workstation behind double NAT. All the machines 
were running Ubuntu 16.04 LTS and a recent version of `docker-ce` (17.09). 
Weave requires 1.12+ Docker.

Install `weave` on all the nodes:
```
curl -L https://git.io/weave -o ./weave
chmod +x /usr/local/bin/weave
sudo mv ./weave /usr/local/bin/
```

Now on each of the nodes, you launch `weave`, telling it to connect to all 
the other machines. Initially, I left my home machine out (since I don’t 
have port forwarding setup). So, on the 4 VPS machines, I ran:
```
weave launch --password <your-secure-password> <node1-ip> <node2-ip> <node3-ip> <node4-ip>
```

The `--password` option along with your chosen password enables encryption 
across all weave connections. Interestingly, it’s safe to tell a node to 
connect to itself. The connection will harmlessly fail (and won’t be retried). 
This really simplifies the launch line, since I used the same weave launch 
command on all the nodes.

My nodes had all inbound ports open. If you have a firewall, then you need to 
open incoming TCP(6783) and UDP(6783,6784).

That’s it!

Now, check the status:
```
weave status
weave status peers
```
It will show that encryption is enabled across all the links and a mesh 
network is setup, with each node connected to each other node. At this point, 
the overlay network is not connected to the host itself, and instead is just 
something to link docker containers with.

The next piece is to simply make the hosts visible on the overlay network. 
Run the following on each node:
```
weave expose
```

That will print out the host’s IP on the overlay network. And `ifconfig weave` 
will also show the IP address. At this point you can ping between hosts using 
the overlay network addresses. You will probably want to add weave expose to 
the boot scripts as the expose effect goes on reboot. The good news is that 
the IP address of the node on the weave network is sticky!

As my home workstation is behind a double NAT, and it’s a pain to do port 
forwarding, it was gratifying to find out that `weave` can easily do NAT 
traversal. So, from my home machine, I just did a weave launch using the same 
password. And, `weave` was able to have my home workstation join the same 
cloud. `weave expose` worked similarly.

At this point we have 5 machines spanning myriad hosting providers, including 
one behind a double NAT, fully connected using an encrypted overlay network.

To have a machine leave the network:
```
weave reset     (on the node you want to leave)
weave forget <leaving-node>    (from any of the other nodes that will remain in the mesh)
```

One issue, ran into was name conflicts when a node joins an existing network. 
I’m guessing whatever algorithm weave uses to derive unique UUIDs is messed 
up due to the proliferation of commodity hardware. In cases of conflicts, 
the node is unable to join the network. The workaround, is to use `ifconfig`
to get the mac address and pass it on the launch line:
```
weave launch --name 02:45:db:ef:ab:0a --password <password> <IP>
```
