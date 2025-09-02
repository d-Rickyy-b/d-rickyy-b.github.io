---
layout: single
classes: wide
title:  "ZeroTier - A smart p2p VPN solution"
date:   2020-10-25 20:00:00 +0200
category: Networking
tags: [VPN, Networking, Decentralization, ZeroTier]
---

# Abstract / tl;dr

ZeroTier is an open source, lightweight P2P VPN solution that helps you connect clients directly. 
It can be used for gaming, file sharing or generally accessing devices behind a NAT. 
It has a whole other focus than regular VPN services but could still be used as a replacement for such.
This article talks about the system and software behind ZeroTier, what to use it for and scratches the surface of the architecture and technical implementation of the software.

# Introduction

If you are like me, you probably have a history of hundreds of hours playing Minecraft. 
Back in the days my friends and me were 12 years old and had no idea what Java is or how a computer works. 
But Minecraft sparked our interest in computers in general. 
In the first few years, we hadn't had any dedicated server to play together. Instead one of our friends hosted the Minecraft server on their computer. 
But since we didn't know anything about "Public" and "Private" IP addresses or "Port Forwarding", we obviously couldn't join. Sooo... we used a tool called "Hamachi" which is a VPN software developed by "LogMeIn". 
Most Minecraft players probably know it.

It allowed us to create a virtual network between all of our computers, so that we didn't need to worry about weird router settings and stuff.
Also it was mentioned and suggested everywhere back then when you wanted to play Minecraft with friends. There were like a gazillion [YouTube tutorials](https://www.youtube.com/watch?v=5zVDzHqwz2M) for it.
However we started gaining knowledge on the topic and eventually even bought our own game server, because it was so annoying to wait for the friend to come online to play.

Now recently I wanted to play a "LAN only" game with my friends over the internet - we weren't able to enter server IPs or stuff - the lobby could only be auto-discovered via LAN. So we thought about installing Hamachi again.
But due to their horrible UX/UI and sometimes even bloated installers, we decided against it.
Next we thought about "Tunngle" (which is somewhat similar to ZeroTier technology wise), but on the one hand it's as bad as Hamachi regarding bloatware and even ads within the application and on the other hand it was shut down in 2018. 
So after some googling we found "ZeroTier One" and gave it a try.
And boy were we happy with it. This article is trying to give you a short introduction to the capabilities of ZeroTier and how it works on a technical note.

# What exactly is "ZeroTier"?

I could probably write pages about this software and keep you busy reading, but instead I decided to copy from their GitHub repo and let the people behind ZeroTier explain it themselves: 
>"ZeroTier is a smart programmable Ethernet switch for planet Earth. It allows all networked devices, VMs, containers, and applications to communicate as if they all reside in the same physical data center or cloud region." [[ZeroTier GitHub repo](https://github.com/zerotier/ZeroTierOne)].

In other words: ZeroTier is a software that enables you to connect multiple devices directly to each other via some kind of VPN. `Speaking of VPNs: our Sponsor NordVPN`... (No, just kidding). But it's not the kind of VPN you'd expect. Even the developers themselves call their product "Virtual Distributed Network (VDN)" instead of VPN. Their GitHub repository proudly calls ZeroTier a "Global Area Networking" solution, which is quite fitting in my opinion.

Note that there is a difference between "ZeroTier" and "ZeroTier One". While the first names the whole system/p2p network, the latter is the name for the client software.

## How does it differ from a regular VPN?

In some regards, ZeroTier (ZT) is pretty similar to a regular VPN. Both are used to connect devices through the public internet via some sort of tunnel, to send traffic from one device to another device securely. Both encrypt the transferred data.

Nowadays, popular VPN services are being used by most people as [glorified proxies](https://gist.github.com/joepie91/5a9909939e6ce7d09e29) to access certain websites they usually can't or to "encrypt their traffic to hide it from the bad guys", or to hide their public IP address from a service. That is not what VPNs were actually intended for. The original intention of VPNs was to provide access to (corporate) networks or to connect remote branch offices to the headquarters.

The biggest difference between them is that ZT is a p2p VPN (or rather VDN as explained earlier) - which means that it connects certain devices directly rather than through a dedicated remote server. There is no single point of failure, after the connection has been established. You can think of it like that: a regular VPN connects networks, or maybe clients with a network. ZT instead connects clients directly which make up the whole network. It acts just as if the devices were connected to the same Ethernet switch.

### Example 1: Consumer Router VPN

My router at home gives me the option to enable a VPN (L2TP/IPSec) endpoint. With that I can connect to my router from the public internet and get an IP address from within my local network. Now my device is part of the entire network. When another, new device (physically) connects to my router, I can instantly connect to it. It looks roughly like displayed in figure 1.

<figure>
  <a href="/assets/img/2020-10-25-zerotier-one/zerotierOneRouterVPN.png">
    <img src="/assets/img/2020-10-25-zerotier-one/zerotierOneRouterVPN.png" alt="Consumer Router VPN (CC BY-ND 4.0)">
  </a>
  <figcaption>Figure 1: Consumer Router VPN (<a href="https://creativecommons.org/licenses/by-nd/4.0/">CC BY-ND 4.0</a>)</figcaption>
</figure>

With ZT I would be directly connected only to those clients which installed the ZeroTier One software and joined the same network as I did.
But I couldn't contact any other device on the router network.

### Example 2: Site2Site VPN

Another thing to compare ZT with is classical Site2Site VPN. Usually companies use that to connect two or more of their branch offices to the headquarters - or to interconnect branched offices. It roughly looks like figure 2.

<figure>
  <a href="/assets/img/2020-10-25-zerotier-one/zerotierOneVPN.png">
    <img src="/assets/img/2020-10-25-zerotier-one/zerotierOneVPN.png" alt="Figure 2: Classical Site2Site VPN (CC BY-ND 4.0)">
  </a>
  <figcaption>Figure 2: Classical Site2Site VPN (<a href="https://creativecommons.org/licenses/by-nd/4.0/">CC BY-ND 4.0</a>)</figcaption>
</figure>

In this case all data must be sent through the gateway servers which then take care of the routing between the branch networks.

### General ZeroTier Architecture

In figure 3 you can see the general architecture of ZT. More on the technical details later. 

<figure>
  <a href="/assets/img/2020-10-25-zerotier-one/zerotierOneTwoClients.png">
    <img src="/assets/img/2020-10-25-zerotier-one/zerotierOneTwoClients.png" alt="Figure 3: Overview of the ZeroTier architecture (CC BY-ND 4.0)">
  </a>
  <figcaption>Figure 3: Overview of the ZeroTier architecture (<a href="https://creativecommons.org/licenses/by-nd/4.0/">CC BY-ND 4.0</a>)</figcaption>
</figure>

**Note:** The figure shows a connection between only two devices. But ZT allows for up to 100 devices for their free tier hosted solution on the same network. If that's not enough for you, you can always host your own root server (called "moon") to connect even more endpoints.
{: .notice--info}

The way ZeroTier works really gives you the ability to create a "Global Area Network" and connect a nearly unlimited amount of clients to the same virtual switch.

With the tools given by ZT you could of course also go ahead and [build yourself some gateway server](https://zerotier.atlassian.net/wiki/spaces/SD/pages/193134593/One+Port+Linux+Bridge) to route all traffic from the ZeroTier interface into the internet and hence creating something very similar to a regular VPN server (see figure 2).

## Wait, so what can I use it for?

There are a lot of use cases for ZeroTier such as:

- Hosting a game server at home (remote LAN party)
  - Useful for LAN only (or cracked) games
- Accessing any device behind NAT directly
  - Access your NAS at home
  - Share a Plex instance with your friends and family
  - Connect via SSH to your home server without opening the port to the whole internet
  - [Use your local PiHole as DNS server from everywhere](https://discourse.pi-hole.net/t/how-to-easily-use-your-pi-hole-outside-of-your-personal-network/18878)
- [Only allow SSH to your server from p2p connected devices](https://breadnet.co.uk/zerotier-cloud-managment/)
- Easily [connect docker containers with each other](https://zerotier.atlassian.net/wiki/spaces/SD/pages/7536656/Running+ZeroTier+in+a+Docker+Container)
- Giving endpoints with changing IPs on the public internet a static IP (within the VDN)

Some of the above use cases are obviously also realizable with a standard VPN software, but often require significantly more configuration and depend on a VPN server that must be active at all times for the connection to keep working. For ZT it's only needed for establishing the connection between two peers and it needs no configuration apart from creating a network & joining it with the clients.

## What was the motivation to create ZeroTier?

The founder of ZeroTier wrote a few excellent blog posts on their reasoning behind creating a p2p VPN like system. Those posts/essays are really, really worth reading. Hence I only want to talk about the key takeaways of those articles and leave the rest for you to read yourself.

Their main point is that the internet became way too centralized. Initially the internet was created as an interconnected, distributed system to better cope with failures and centralized outages. When a certain route between two ISPs goes down, the routing protocols will quickly redirect all traffic over another connection, keeping the internet alive.

Now apart from centralized **services** such as Dropbox, Facebook etc. we see the effects of centralization now more than ever: A large number of sites hosted by major hosting companies or using DDoS protection from vendors like Cloudflare will suffer from connection issues, when the hoster is having outages or other issues. On July 2, 2019 Cloudflare suffered a major outage that made to 12 million websites go offline for 27 minutes.

Cloudflare (as a reverse proxy) is being [used by ~14.8%](https://w3techs.com/technologies/details/cn-cloudflare) of all known websites. That's more than 1/8th of the entire internet (meaning websites).

Don't get me wrong, there are indeed good reasons for centralization (and even for the usage of Cloudflare). But at the same time centralization introduces new risks and issues (technical as non-technical ones) to cope with. 
>"All that centralization comes at a cost: monopoly rents, mass surveillance, functionality gate keeping, uncompensated monetization of our content, and diminished opportunities for new entrepreneurship and innovation."  
[[Adam Ierymenko](https://adamierymenko.com/decentralization.html)]

Also building decentralized systems is plain hard to do. That's basically why ZeroTier was created. To provide users and developers with tools to easily ("with zero configuration") connect devices into a distributed network.

Go ahead and read the blog post ["Op-ed: Internet centralization is not a conspiracy"](https://zerotiernetworks.tumblr.com/post/58157836374/op-ed-internet-centralization-is-not-a-conspiracy) and the essay ["Decentralization: I Want To Believe"](https://adamierymenko.com/decentralization.html) by Adam Ierymenko - the founder of ZeroTier. Especially the latter is a **very good read**.

# Technical insights

If you only wanted to know what ZeroTier is and how it can be used, you can finish reading here. From this paragraph on I will only explain the technology behind ZT.

Whilst working with ZT I barely got in contact with the underlying technology. It was so easy and simple to set up that I started wondering about how this software actually works. Following my interest I decided to take a look under the hood. 
There is a lot to talk about and I really don't want to bore you out with pesky details, so I try to keep it as short as necessary to understand it. I'll focus on the architecture only. Also a quick reminder: the following is my understanding of it - I might have gotten something wrong.

## P2P Connection

Since most devices on the internet are somewhere behind a NAT device (e.g. your router), there is no easy way to initiate a direct communication to a device from outside the local network. 
To achieve a direct communication of peers, ZeroTier came up with an interesting solution. They divided the connection into two parts:

- VL1 - p2p component (encryption, direct communication)
- VL2 - virtual Ethernet component (authorization, access control, network rules)

**VL1** is designed to be zero-configuration. Users don't need to write configuration files or fiddle with IP addresses or similar. VL1 is used to connect two endpoints directly to each other (similar to OSI **L**ayer **1** - hence the name V**L1**) with the help of the Root Servers. It is the peer2peer component in ZeroTier, that creates the virtual network itself. It also implements technologies such as [UDP Hole Punching](https://en.wikipedia.org/wiki/UDP_hole_punching) which are being used by ZT to circumvent NAT. And as mentioned in the introduction of this post, NAT is the main reason why it is so hard for inexperienced users to host a gaming server from home.

Then there is **VL2** which is the Ethernet virtualization layer (similar to OSI **L**ayer **2** - hence the name V**L2**). 
> VL2 is a [VXLAN](https://en.wikipedia.org/wiki/Virtual_Extensible_LAN)-like network virtualization protocol with SDN management features. It implements secure VLAN boundaries, multicast, rules, capability based security, and certificate based access control.  
[[ZeroTier Manual](https://www.zerotier.com/manual/#2_2)].

This is the component where the actual Ethernet packets are sent over.
Another interesting fact you should know is that clients identify themselves via cryptographic identifiers.

## Root Servers (Planet/Moons)

For the initial connection setup between peers via VL1 we need some known third party mediator that both peers contact to exchange information about the other peer(s). Once the information has been exchanged, the peers knows how to contact each other and can establish a connection between them (see figure 3). If a connection couldn't be established, the root server can also be used as a traffic relay (only as last resort). 
That way a connection between two clients can always be established, even if regular NAT traversal techniques don't work.

And even in case of relaying traffic, the root servers can't read any data from the traffic, due to the e2e encryption of the ZeroTier VL1 layer.
But in the best case scenario the clients don't need to send their actual traffic to the ZeroTier root server at all because they managed to establish a direct p2p connection.

ZeroTier runs 12 root servers which IP addresses are known to each client. They are called "planets". But you can also [set up a custom "moon" server](https://www.zerotier.com/manual/#4_4).
>A moon is just a convenient way to add user-defined root servers to the pool. Users can create moons to reduce dependency on ZeroTier, Inc. infrastructure or to locate root servers closer for better performance.  
[[ZeroTier Manual](https://www.zerotier.com/manual/#2_1_1)]

After the connection has been established, the clients contact the network controller.

## Network Controller

> Every ZeroTier virtual network has a network controller responsible for admitting members to the network, issuing certificates, and issuing default configuration information.  
[[ZeroTier Manual](https://github.com/zerotier/ZeroTierOne/tree/master/controller)]

The controller distributes the network configuration among the clients. It also uses cryptographic methods to authenticate devices. It's the configuration manager of virtual networks (VL2). A single controller can in theory handle up to 2^24 networks (16.777.216) and even more devices. If running your own controllers, you can deploy multiple instances to handle even more networks.

To outline the purposes of and differences between the root servers and controllers even more: 
> Root servers are connection facilitators that operate at the VL1 level. Network controllers are configuration managers and certificate authorities that belong to VL2. Generally root servers don’t join or control virtual networks and network controllers are not root servers, though it is possible to have a node do both.  
[[ZeroTier Manual](https://github.com/zerotier/ZeroTierOne/tree/master/controller)]

**Note:** In figure 3 there is no visible link from the two clients to the network controller. This was left out intentionally to keep the figure a bit less complex. But of course both clients need to contact the controller at least once via VL1.
{: .notice--warning}

As said previously, you can also [run your own controller](https://github.com/zerotier/ZeroTierOne/tree/master/controller) for independence from ZeroTier.

## ZeroTier Central

Hosted webGUI for the Network Controller with a simple user interface to configure your networks, authorize clients and configure the rule set for the rule engine. Accessible at [my.zerotier.com](https://my.zerotier.com/). From what I found, this webGUI is not part of the controller software. Hence someone created a web front end for self-hosted controllers named [ztncui](https://github.com/key-networks/ztncui).

## Network Rules Engine

After joining a virtual network, there is no (hardware) firewall in place to protect your devices from malicious packets of peers.
But ZeroTier got you covered. As a network maintainer you can utilize the "Rules Engine" of ZT.
Those rules are pretty similar to regular firewall rules of popular vendors.

The rules are distributed by the network controller and are enforced on the sender and receiver side. So an attacker would need to compromise both sides to circumvent the rules engine.

There is a huge difference from regular firewall rules. The Rules Engine of ZT is stateless, meaning it lacks connection tracking.
Instead of state tracking they implemented capability-based security and device tagging. 
That way you can give granular permissions to devices or groups of devices. 
Which means that you can segment your virtual network into groups and permit or deny certain traffic from and to other groups. 
And since capabilities and tags are cryptographically signed, we can be certain that a peer claiming to have those capabilities/tags actually has them. Because they can prove it. 

Also that means that there is no way to spoof (or "steal") capabilities/tags by simply being a member of the network. You'd need to compromise the client to achieve this.

```text
# A tag we might use to group devices together as e.g. members of the same
# company department.
tag department
	id 1000 # Tags have arbitrary 32-bit IDs like capabilities
	enum 100 accounting
	enum 200 sales
	enum 300 finance
	enum 400 legal
	enum 500 engineering
;

# Allow only IPv4 (and ARP) and IPv6. Drop other traffic.
drop
	not ethertype 0x0800 # ... and ...
	not ethertype 0x0806 # ... and ...
	not ethertype 0x86dd # (multiple matches in an action are ANDed together!)
;

# A variant on a normal TCP whitelist to allow ssh only between
# members tagged as part of the same department. Could also make
# this a macro but we only need it once here.
accept
	tdiff department 0 # no difference
	ipprotocol 6
	chr tcp_syn
	not chr tcp_ack
	dport 22 22 # ports are ranges, in this case it's a range of size 1
;
```

The above example is an excerpt taken from [this gist](https://gist.github.com/adamierymenko/7bcc66b5f7627699236cda8ac13f923b). It's a simple ruleset written in the "Rule Definition Language" developed by ZeroTier.

All in all the rules engine is a very powerful tool to restrict access even down to single devices. It might not be as capable compared to a stateful firewall in some cases, but it gives more than enough options for basic firewalling or basic network traffic restriction.

## Security implications

As explained earlier, joining a ZeroTier network is basically the same as if you'd connect these same devices to one Ethernet switch. So you should be aware of what that means from a security point of view.
Since you are connected directly to other devices - depending on your use case even devices from strangers - you should ensure that the firewall of your OS is enabled (e.g. Windows Firewall, iptables).
There is no extra NAT router or firewall (apart from the Network Rules Engine) blocking packets within the network.
For example: If you are running an SSH server on the ZeroTier network interface, other clients could try to log into your system. Make sure to restrict access where not needed.

If you are the network maintainer, make sure to restrict access via the ZT Rules Engine to prevent access on the network level already.
And last but not least: Always keep your software up to date. Outdated software can contain vulnerabilities that in some cases can be abused more easily from adjacent networks.

## SDK

ZeroTier also provides us with a lightweight library named ["libzt"](https://www.zerotier.com/manual/#5) that one can use to implement in their own projects. That way it should be a lot easier to create a distributed application.

# Alright, how to set it up?

Setting up ZeroTier One is very easy:

1. Create an account on [my.zerotier.com](https://my.zerotier.com/)
2. Create a network
3. [Download](https://www.zerotier.com/download/) the ZeroTier One software
4. Join the network

If you are more of a "visual person", I recommend you [this video by Lawrence Systems](https://www.youtube.com/watch?v=ZShna7v77xc), explaining how to setup ZeroTier One with two windows computers. But setting it up on Linux, MacOS, FreeBSD, your NAS, Android or iOS is just as simple.

# Final notes

I truly think that we all could benefit from a bit more decentralized IT world with less giant companies owning huge services but more smaller, distributed services. And in my opinion ZeroTier is currently the easiest way to achieve this goal.

I want to close this post with a huge "thank you" for reading this far. I hope you learned something from reading this post. If you got any questions or feedback, feel free to reach out for me.

All the images used in this article are my own creations. I used the tool yEd to create them. You are free to use these images and the [yEd source file](/assets/files/2020-10-25-zerotier-one/zerotierOne.graphml) under the license terms of [CC BY-ND 4.0](https://creativecommons.org/licenses/by-nd/4.0/).
