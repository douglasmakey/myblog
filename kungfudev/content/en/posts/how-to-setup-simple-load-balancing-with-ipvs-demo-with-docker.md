---
title: "How to setup simple load balancing with IPVS, demo with docker"
date: 2020-04-06
tags: ["k8s", "ipvs", "network", "linux"]
cover:
    image: "https://dev-to-uploads.s3.amazonaws.com/i/awego9xodgmruk8lm3am.png"
---

A few days ago, I was reading about the Kubernetes network model, especially about `services` and the `kube-proxy` component, and I discovered that `kube-proxy` has three modes, which are `userspace`, `iptables` and `ipvs`.

The `userspace` mode is too old and slow, nowaday nobody recommends to use it, the `iptables` mode is the default mode for `kube-proxy` with this mode `kube-proxy` use iptables rules to forward packets that are destined for services to a backend for that services, and the last one is `ipvs` I did not know what it was so I read about it.

### What is IPVS?
*IPVS (IP Virtual Server) is built on top of the Netfilter and implements transport-layer load balancing as part of the Linux kernel.*

*IPVS is incorporated into the LVS (Linux Virtual Server), where it runs on a host and acts as a load balancer in front of a cluster of real servers. IPVS can direct requests for TCP- and UDP-based services to the real servers, and make services of the real servers appear as virtual services on a single IP address.*


That means **IPVS is a Linux kernel load balancer over layer 4**, if you don't know what is the difference between LB L4 and LB L7 there is a good explanation [Here](https://freeloadbalancer.com/load-balancing-layer-4-and-layer-7/).

**LB L4**

*At Layer 4, a load balancer has visibility on network information such as application ports and protocol (TCP/UDP). The load balancer delivers traffic by combining this limited network information with a load balancing algorithm such as round-robin and by calculating the best destination server based on least connections or server response times.*

So in this layer, you are not parsing the data in the packages, so you don't know what's inside, for instance, if you are using LB 4 and receive an HTTP request at this layer you can't see the `path` or the body o headers of this request so you cant take a smart decision based on this.

**LB L7**

*At Layer 7, a load balancer has application awareness and can use this additional application information to make more complex and informed load balancing decisions. With a protocol such as HTTP, a load balancer can uniquely identify client sessions based on cookies and use this information to deliver all a clients requests to the same server. This server persistence using cookies can be based on the server’s cookie or by active cookie injection where a load balancer cookie is inserted into the connection. Free LoadMaster includes cookie injection as one of many methods of ensuring session persistence.*


## Simple Demo

Now you know what is IPVS, we are going to make an ultra-simple demo using docker to have an LB with IPVS between two containers.

The first thing that we need is the CLI tool for interacting with the IP virtual server table in the kernel.

ipvsadm - Linux Virtual Server administration [ipvsadm](https://linux.die.net/man/8/ipvsadm).


```bash
sudo apt-get install -y ipvsadm
```

### Create the virtual service.

Now, we can use the CLI to create a new virtual service:

```bash
ipvsadm COMMAND [protocol] service-address [scheduling-method] [persistence options]
```

If we see in the documentation of [ipvsadm](https://linux.die.net/man/8/ipvsadm), we will see that using the flag `-A` we indicated "Add a virtual service", and the flag `-s` is for the scheduling-method, first we will try with `rr` that means  Round Robin, we have different options such as: wrr - Weighted Round Robin, lc - Least-Connection, lblc - Locality-Based Least-Connection and more.

So we are creating the virtual service for the address 100.100.100.100:80 using Round Robin as scheduling-method.


```bash
sudo ipvsadm -A -t 100.100.100.100:80 -s rr
```

### Create two docker container

We are going to use the image `jwilder/whoami` for our containers, this image just returns the container's id.


```bash
$ docker run -d -p 8000:8000 --name first -t jwilder/whoami
cd977829ae0c76236a1506c497d5ce1628f1f701f8ed074916b21fc286f3d0d1

$ docker run -d -p 8001:8000 --name second -t jwilder/whoami
5886b1ed7bd4095cb02b32d1642866095e6f4ce1750276bd9fc07e91e2fbc668
```

Then, we are going to get IP of these containers, using `docker inspect`

```bash
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' first
172.17.0.2

$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' second
172.17.0.3

```

Using `curl` to one of these containers, we will see the container_id.

```bash
$ curl 172.17.0.2:8000
I'm cd977829ae0c
```

### Add the IPs to the virtual service.

We have the containers' IP, so we are going to add these IP to the virtual service using `ipvsadm` with the flags `-a` to add a server to the virtual service that we specified using `-t` and `-m` to use masquerading (network access translation, or NAT).

```bash
$ sudo ipvsadm -a -t 100.100.100.100:80 -r 172.17.0.2:8000 -m
$ sudo ipvsadm -a -t 100.100.100.100:80 -r 172.17.0.3:8000 -m
```

We can use the `ipvsadm` to list the virtual services with its servers.

```bash
$ ipvsadm -l
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  100.100.100.100:http rr
  -> 172.17.0.2:8000              Masq    1      0          0
  -> 172.17.0.3:8000              Masq    1      0          0
```

If you added a wrong server, you can remove that server with `-d` flag.

```bash
$ ipvsadm -d -t 100.100.100.100:http -r 172.17.0.3:8000
```


Now our service can make loadbalacing over L4 using Round Robin as algorithm to balance.

```bash
$ curl 100.100.100.100
I'm 5886b1ed7bd4

$ curl 100.100.100.100
I'm cd977829ae0c

$ curl 100.100.100.100
I'm 5886b1ed7bd4

$ curl 100.100.100.100
I'm cd977829ae0c
```

As you can see, doing load balancing with `IPVS` is pretty straightforward.

In **K8S** one of the advantages of choosing `IPVS` mode for `kube-proxy` instead of `iptables` is that `IPVS` is a Linux kernel feature that is designed for load balancing, it has multiple different scheduling algorithms such as round-robin, shortest-expected-delay, least connections and more, also it has an optimized look-up routine  O(1) based on a hash table data structure rather than a list of sequential rules O(n) "iptables adds the rules in a sequential chain that  grows roughly in proportion to the number of services and number of backend pods behind each service)."