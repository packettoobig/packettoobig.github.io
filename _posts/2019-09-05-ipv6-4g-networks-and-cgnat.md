---
title: IPv6, 4G Networks and CG-NAT
slug: ipv6-4g-networks-and-cgnat
redirect_from: 'ipv6-4g-networks-and-cgnat/'
date_published: 2019-09-05T13:21:43.000Z
date_updated: 2020-10-31T18:46:38.000Z
tags: Thoughs, IPv6
excerpt: My little story about CGNAT and IPv6 on mobile networks
---

*Disclosure : I dislike NAT in general, and I think the IPv6 migration should have ended  10 years ago, so that we could all have a real end-to-end host connecticity and therefore a real internet.*

Yesterday I was talking with a coworker about IPv6 deployment on mobile networks. I noticed that I still had no access to the IPv6 world from my phone with my provider. On the other hand, his provider was giving him a proper IPv6 connectivity.

However, there was something in common for both of our providers. And it was that our local device IPv4 address was part of the `100.64.0.0/10` address block (the one normalized in [RFC 6598](https://tools.ietf.org/html/rfc6598)). For example, mine was `100.81.212.60/25`.

I did some research and I still can't figure why end user devices are getting IPv4 addresses from this pool, since we should be getting [RFC 1918](https://tools.ietf.org/html/rfc1918) IPv4 addresses instead.

My understanding of CG-NAT (also called NAT444) is represented in this nice diagram that I found [here](https://reggle.wordpress.com/2012/07/15/rfc-6598-carrier-grade-nat-explained/) :

![nat444-1](/assets/ipv6-4g-networks-and-cgnat/nat444-1.png)

In theory my phone (and my colleague's) should be getting regular RFC1918 addresses then, so why is that ?

You may think that I have an answer, but I actually don't, so feel free to tell me is you have any information about that, and I'll update this post.
