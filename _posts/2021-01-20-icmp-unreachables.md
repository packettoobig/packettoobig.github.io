---
title: ICMP Unreachables
slug: icmp-unreachables
redirect_from: 'icmp-unreachables/'
date_published: 2021-01-20T11:10:00.000Z
date_updated: 2021-01-29T07:02:57.000Z
tags: Security, IPv6, IPv4
---

Generally speaking, the Internet is doing well recently:

- Most networks are running over Ethernet now.
- IPv6 is gaining more and more traction (Of course we are not done yet, but still, the figures are getting better every year)
- TLS variants are now more prevalent than its non-encrypted counterparts (SSH, HTTPS, ...).
- QUIC is really promising.

But we still have some challenges to solve, and one of them is [Path MTU Discovery](https://en.wikipedia.org/wiki/Path_MTU_Discovery), that is still kind of broken.

The two main reasons are :

- Configuration errors
- Stupid defaults

## Configuration errors

### Unreachable

If you are running Cisco IOS/IOS-XE or NX-OS, **do not ever do this**

    interface <name>
      no ip unreachables
      no ipv6 unreachables
    

On IOS-XR, the equivalent would be :

    interface <name>
      no ipv4 unreachables
      no ipv6 unreachables
    

If you read the wikipedia page, you already know why you should not do this.

For you, lazy people, the short explanation is that each interface in the path of a packet must be able to signal that this packet cannot reach its destination for some reason.

In our case, we want to make sure that each interface can signal that a packet is too big, and that it should then be fragmented (ICMP Type 3, code 4 or ICMPv6 type 2).

### ACL

You probably already know that Cisco IPv4 ACL end up with a "default deny", which is fine for L3 ACLs.

The same thing applies to IPv6 ACLs, with the exception of Neighbor Discovery messages indeed.

However, when you start dealing with L4 (extended) ACLs, you might want to only allow ports 80 and 443 for example.

Your ACL would look something like this :

    ipv6 access-list ACL-ALLOW-WEB-V6
     permit tcp any any eq www
     permit tcp any any eq 443
    

It makes sense right ? We only want a web access to the internet !

Well, **this is totally wrong**.

What if the users are using VPN encapsulation ?

What if any other link in the path has a lower MTU for some reason (like IPv4 to IPv6 transition methods, or PPP) ?

If you realy want to do ACLs, you should really do this instead:

    ipv6 access-list ACL-ALLOW-WEB-V6
     permit tcp any any eq www
     permit tcp any any eq 443
     permit icmp any any
    

And you have a really strict security policy, the [RFC4890](https://tools.ietf.org/html/rfc4890) compliant ACL would probably look something like this:

    ipv6 access-list ALLOW-WEB
     permit tcp any any eq www
     permit tcp any any eq 443
     permit icmp any any unreachable
     permit icmp any any packet-too-big
     permit icmp any any hop-limit
     permit icmp any any next-header
     permit icmp any any parameter-option
     permit icmp any any reassembly-timeout
     permit icmp any any header
    

However I would still recommend you to at least allow echo and echo-reply as well, plus it is required in IPv6 for Teredo.

Also, you should allow UDP port 443, because QUIC is coming soon :)

## Stupid Defaults

I can't really provide you with a config for every brand of network equipment, but the first thing you should do, especially with a Firewall, is to create an `allow icmp any any` rule (or equivalent) before doing anything else.

If you have strict security policies, you may do it slightly differently, but you must still allow the following ([RFC4890](https://tools.ietf.org/html/rfc4890)):

- Destination Unreachable
- Packet Too Big
- Time Exceeded
- Parameter Problem

Please be aware, however, that doing so would contribute to the [ossification](https://en.wikipedia.org/wiki/Protocol_ossification) of the Internet, and you will be the bad guy here :).
