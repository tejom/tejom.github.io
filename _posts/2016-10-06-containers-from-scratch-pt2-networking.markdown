---
layout: post
title:  "Containers From Scratch Pt.2 Networking"
date:   2016-10-08
categories: c linux containers docker networking
---

The next thing to do was do get my containers to do some useful stuff on the Internet. I wanted my containers to ping each other, connect out to the Internet and listen on ports while being fully isolated from the main system. I had to resort to scripting to get this working simply. A lot of the networking I'm sure is really interesting but would have taken me off track for the goal here.

I started with isolating the network namespace.  Add the flag `CLONE_NEWNET` to clone. If you run the container and check the network configuration there should only be the loopback.

If the network namespace is isolated it would seem unintuitive to be able to have connections. veth devices can be used to make the connection between namespaces. The best description I read was to think of them as a pipe between namespaces. This clicked for me when I thought of a veth pair as an actual ethernet cable.

```
    -------------------------
    |ns1        |       ns2 |
    | |veth0|<==|==>|veth1| |
    |           |           | 
    -------------------------
```

I added two functions, create_peer and network_setup, that will set this up.

These are two helpful guides on setting up veth pairs and network namespaces. I implement what these explained.

<a href="http://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/">http://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/</a>
<a href="https://blog.yadutaf.fr/2014/01/19/introduction-to-linux-namespaces-part-5-net/">https://blog.yadutaf.fr/2014/01/19/introduction-to-linux-namespaces-part-5-net/</a>

Main will call create_peer, which will do the initial configuration of the veth peer.

``` c
int create_peer()
{
        char *id = (char*) malloc(4);
        char *set_int;
        char *set_int_up;

        rand_char(id,4);    
        printf("id is %s\n",id);
        asprintf(&set_int,"ip link add veth%s type veth peer name veth1",id);
        system(set_int);
        asprintf(&set_int_up,"ip link set veth%s up",id);
        system(set_int_up);

        free(id);
        return 0;
}
```

This code sets up some strings to call with system(). The veth interface has a random name so multiple containers can exist with veth pairs. veth1 is unique to the container so the same doesn't matter.(Docker renames this eth0) The first command creates the veth pair. The second brings up the interface that will stay in host's namespace. You could give the host side interface an ip address here, along with some other work that will be mentioned the next few steps and connect to the internet.

The next function called is network_setup

``` c
int network_setup(pid_t pid)
{
        char *set_pid_ns;
        asprintf(&set_pid_ns,"ip link set veth1 netns %d",pid);
        system(set_pid_ns);
        return 0;
}
```

This is simple. It places veth1 into the network namespace of the child pid.

main is modified like this:

``` c
    create_peer();
    pid_t pid = clone(child_exec,c_stack, flags ,args);
    if(pid<0)
        fprintf(stderr, "clone failed %s\n", strerror(errno));
    network_setup(pid);
```

child_exec needs to be modified to set up its new interface. I also modified main to take an ip address as an argument and pass that along to child_exec when clone is called. That is the new variable ip.

``` c
    system("ip link set veth1 up");
    char *ip_cmd;
    asprintf(&ip_cmd,"ip addr add %s/24 dev veth1",ip);
    system("route add default gw 172.16.0.100 veth1");
```

This brings up veth1 in the container and gives it an ip address. The default gateway can be the ip address of the hosts veth interface for the pair. It is going to be set to the ip address of the bridge that will be set up. 
If you gave the host veth interface an ip address earlier you can enable Internet connections with iptables forward rule and nat.

``` bash
sudo iptables -A FORWARD -i enp0s3 -o veth -j ACCEPT
sudo iptables -A FORWARD -o enp0s3 -i veth -j ACCEPT
iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -j MASQUERADE
```

enp0s3 is the main interface in virtualbox.

Now we have basic connectivity that's isolated in the container.
More complex networking is more interesting. A lot of the work is done in the shell now.

Having containers communicate with each other can be done with a bridge in the host's namespace. Having a port in the container available on the host is done with iptables and port forwarding.
I got a lot of insight into setting this up from here. <a href="http://54.71.194.30:4017/articles/networking/">http://54.71.194.30:4017/articles/networking/</a>

The next couple steps will be:

* Setting up a bridge in the host network namespace
* Attaching the host side of the veth pair to the bridge
* Setting up routing, nat, and ip addresses
* Setting the containers default gateway to the bridge's IP
* Configuring iptables to forward ports

The containers will all use their veth pair to communicate and the bridge will act as a router through rules in iptables. 

Setting up a bridge is the next step.
I'm using 172.16.0.0/24 for the examples.

``` bash
brctl addbr br0
ip addr add dev br0 172.16.0.100/24
ip link set br0 up
```

Create a bridge named br0,give it an ip address, bring up the interface.

And set up initial nat.

``` bash
sudo iptables -A FORWARD -i enp0s3 -o br0 -j ACCEPT
sudo iptables -A FORWARD -o enp0s3 -i br0 -j ACCEPT
iptables -t nat -A POSTROUTING -s 172.16.0.0/16 -j MASQUERADE
```

We need to modify create_peer to add the hosts veth interface to the bridge.

``` c
        asprintf(&add_to_bridge,"brctl addif %s veth%s",BRIDGE,id);
        system(add_to_bridge);
```

This can be after bringing the interface up and before returning. Make sure this interface doesn't have an ip address or connections won't work properly. `brctl show br0` should show the created interface is attached to it.

In child_exec modify 
`system("route add default gw 172.16.0.100 veth1");`
to set the default gateway to the ip address of the bridge.

At this point if you bring up two containers they should be able to connect to each other and connect out to the intenet.

The rest of the work to forward ports is done in iptables. This is like running `docker run -p ##:##` or something similar.

In the container run something like `nc -l -p 8000`

There are three main commands to do this:

``` bash
iptables -I PREROUTING 1 -t nat -p tcp --dport 80 -j DNAT --to 172.16.0.30:8000
iptables -A OUTPUT -t nat -p tcp --dport 80 -j DNAT --to 172.16.0.30:8000
iptables -A FORWARD -p tcp -d 172.16.0.30 --dport 8000 -j ACCEPT
```

The first command sends all the requests to the containers ip at the port were listening on.
`curl <ip address of the host>` and you should have connected to the container.

Note: Somehow between upgrading from the rc5 kernel I was using to the released 4.8 kernel this stopped working for local connections. The interface used when using curl to a local ip address switched from the main interface to the loopback.

That is about all of the major basic features of a container covered and implemented. 

Check the final code here <a href="https://github.com/tejom/container">https://github.com/tejom/container</a>

How to run:

This is fairly simple to run if you would like to try it.

First clone the repo and run `make`. This will create a executable called main.

The program takes to sets of arguments, an ip address for the container and a command to execute.

`sudo ./main 172.16.0.30 bash`

Will run bash in the container.

`sudo ./main 172.16.0.30 nc -l -p 8000`

Will run the nc command with the arguments.



