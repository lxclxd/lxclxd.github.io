---
layout: post
title: 'LXC with public (external) IP address'
date: 2017-02-07 20:25:06
description: How to make container accessible outside of the host
comments: true
tags:
 - lxc
 - network
---

## Tested on:

* ubuntu 16.04
* two network cards: eno1, eno2
* dhcp server in the network

We want each container to have its own dedicated IP address, being accessible from outside world.

**It's a good idea to disable AppArmor and firewall when testing new network configuration.**

> I use a physical server. No wifi. No host virtualization. 
> It may or may not work for you. Because of your hardware. Or your software. Or bad luck.

Take one step at a time.
Start with the host (macvlan/bridge). See if it's working. If it is - start playing with the container.

## One way or the other
Your host network connection needs to be configured either as a **bridge** or **macvlan**.

In a few words: try to make macvlan work. If it doesn't - use a bridge.

A bridge is basically a network switch. It learns MAC addresses, uses STP to prevent loops and so on.
A macvlan interface is much less complex. It enables to add multiple MAC addresses to a single physical interface.
It's faster. It uses less CPU. It's perfect for containers. 

## Network interface bonding

We have two network interfaces: eno1 (with mac address 00:11:22:33:44:55) and eno2 (66:77:88:99:aa:bb).
DHCP server has been configured to assign the right IP address to the first one.

If you have (or use just) one network card - you can skip this part. Just treat the bond0
interface as your network card (eth0, eno1, whatever you got).

In our environment network cards are connected to different switches. We'd like to make sure that if
one of them fails we won't lose network connection. Bonding (link aggregation) with the active-backup
policy provides this kind of fault tolerance.

Here's our initial **/etc/network/interfaces**:

{% highlight config %}
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual
bond-master bond0
bond-primary eno1

auto eno2
iface eno2 inet manual
bond-master bond0

auto bond0
iface bond0 inet dhcp
hwaddress ether 00:11:22:33:44:55
bond-mode active-backup
bond-miimon 100
bond-slaves none
{% endhighlight %}

Your bonding configuration may be slightly different (depending on your bonding protocol).
Just make sure it's working as expected.

From now on we assume that bond0 is our network interface on the LXC host.

## Macvlan interface in two steps

In the first step we'll create a macvlan interface on top of our bonding interface (bond0).
Once finished we'll have eno1, eno2, bond0 and macvlan0.
An IP address will be assigned only to the last one (but all of them will be "in use").
Here we go:

{% highlight config %}
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual
bond-master bond0
bond-primary eno1

auto eno2
iface eno2 inet manual
bond-master bond0

auto bond0
iface bond0 inet manual
hwaddress ether 66:77:88:99:aa:bb
bond-mode active-backup
bond-miimon 100
bond-slaves none
address 0.0.0.0

auto macvlan0
iface macvlan0 inet dhcp
pre-up ip link add link bond0 name macvlan0 address 00:11:22:33:44:55 type macvlan mode bridge
{% endhighlight %}

We keep the bond0 interface because it provides fault tolerance. However we changed its 
IP configuration to "manual". In my testing environment I had to assign an IP address
of 0.0.0.0 - without it I wasn't able to make macvlan work.
As you can see macvlan0 uses bond0 (so eno1 and eno2) to talk to the world.

One last thing thing - in our example there's a DHCP server in the network expecting mac
address of 00:11:22:33:44:55 so we moved that mac from bond0 to macvlan0, leaving the bonding
interface with the mac address of the second physical interface (eno2). 

Restart your server and see if everything works as expected.

*Note: if you have (or use) just one network interface (eno1) and you don't use DHCP - your 
configuration should look more or less like this (I'm guessing):*

{% highlight config %}
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual
address 0.0.0.0

auto macvlan0
pre-up ip link add link eno1 name macvlan0 address 00:11:22:33:44:55 type macvlan mode bridge
iface macvlan0 inet static
address 192.168.1.2
netmask 255.255.255.0
{% endhighlight %}

*We basically create a macvlan interface with a random mac address and assign the right
IP address to it. In case of any problems I'd start with assigning some random mac to the
eno1 interface and move its original mac to the bonding interface (still guessing).*

Once we have our LXC host with the macvlan interface working we can move to the second
step - container configuration. In our example the container name is "ubuntu1" so its 
(default) configuration path is **/var/lib/lxc/ubuntu1/config**. Oh, before we start it's a good idea
to execute the **lxc-checkconfig** command to make sure that support for macvlan interface
is 'enabled'.

In the container config file (/var/lib/lxc/ubuntu1/config) we need to create new entries
**lxc.network....** removing old ones. Here we go:

{% highlight config %}
# This is our old, default network config
# lxc.network.type = veth
# lxc.network.link = lxcbr0
# lxc.network.flags = up
# lxc.network.hwaddr = cc:dd:ee:ff:00:11

# and the new one
lxc.network.type = macvlan
lxc.network.macvlan.mode = bridge
lxc.network.link = macvlan0
lxc.network.flags = up
lxc.network.hwaddr = cc:dd:ee:ff:00:11
{% endhighlight %}

We're using our host's macvlan0 interface to provide the container with "direct" access to the network.
We're also creating a random mac address for our container. That's it.

Once our container is running it has a network interface that can be configured in 
a normal, regular way - with an IP address assigned manually or by using DHCP server
(in our case (Ubuntu) via /etc/network/interfaces), like any other physical host.

## Bridge interface in two steps

In the first step we create a bridge interface with only bond0 attached to it. 
Once finished we'll have eno1, eno2, bond0 and br0.
An IP address will be assigned only to the last one (but all of them will be "in use").
Before you start - make sure you have the **bridge-utils** package installed.
Here we go:

{% highlight config %}
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual
bond-master bond0
bond-primary eno1

auto eno2
iface eno2 inet manual
bond-master bond0

auto bond0
iface bond0 inet manual
bond-slaves none
bond-mode active-backup
hwaddress ether 00:11:22:33:44:55

auto br0
iface br0 inet dhcp
bridge_ports bond0
bridge_stp off
{% endhighlight %}

Restart your server and see if everything is working as expected.
Use the **brctl show** command to see if bond0 has been attached to the bridge.

*Note: if you have (or use) just one network interface (eno1) and you don't use DHCP - your
configuration should look more or less like this (I'm guessing):*

{% highlight config %}
auto lo
iface lo inet loopback

auto eno1
iface eno1 inet manual

auto br0
iface br0 inet static
bridge_ports eno1
bridge_stp off
address 192.168.1.2
netmask 255.255.255.0
{% endhighlight %}

*The br0 interface takes over host's IP configuration and the **brctl show** command 
says that the eno1 interface is the only member of the bridge.*

Our second step - container configuration (**/var/lib/lxc/ubuntu1/config**) - is really simple: 
just replace the default **lxcbr0** with **br0**:

{% highlight config %}
# use br0 instead of lxcbr0
lxc.network.type = veth
# lxc.network.link = lxcbr0
lxc.network.link = br0
lxc.network.flags = up
lxc.network.hwaddr = cc:dd:ee:ff:00:11
{% endhighlight %}

Now you can start up the container and configure its connection as usual - by modifying the 
/etc/network/interfaces file. On the host side: every time a container starts -
a new, virtual network interface joins the bridge (**brctl show**). That's it.
