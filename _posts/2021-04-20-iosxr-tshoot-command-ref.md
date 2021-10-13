---
title: IOS XR quick troubleshooting command reference
slug: iosxr-tshoot-command-ref
redirect_from: 'iosxr-tshoot-command-ref/'
date_published: 2021-04-20T07:27:43.000Z
date_updated: 2021-04-20T07:27:43.000Z
tags: Cisco
excerpt: A nice and handy IOS XR command reference
---

I've done it for Junos, so I figured out it might be useful to have one for IOS XR as well !
I will try to keep this one up-to-date as well as the Junos one !

## Interfaces

Display the interface part of the configuration
`show run interface`

Check L1 and L2 status
`show interfaces <interface-name>`

SFP/Transceiver/Fibre checking
`show controllers <interface-name>`

List all interfaces with their IP
`show ipv4 interface brief`
`show ipv6 interface brief`

List all interfaces with their IP and VRF
`show ipv4 vrf all interface brief`
`show ipv6 vrf all interface brief`

## LACP

Show the status of all LACP interfaces:
`show bundle brief`

## ARP and NDP

Display the ARP table (ipv4)
`show arp`

Display the ND table (ipv6)
`show ipv6 neighbors`

## Hardware troubleshooting

Temperature, fans and power
`admin show environment temperature`
`admin show environment fan`
`admin show environment power`

Serial numbers and PID
`show inventory`

## System troubleshooting

Display the configuration
`show running-config`

Uptime and version
`show version`

NTP
`show ntp status`

NTP servers
`show ntp associations`

Show DNS servers
`show hosts`

Currently logged-in users
`show users`

Last configuration changes with time and date
`show configuration commit list`

## Routing protocols troubleshooting

### IS-IS

Show the IS-IS part of the configuration
`show run router isis`

Show on which interfaces is IS-IS running
`show isis interface brief`

Show on which interface IS-IS actually established an adjacency
`show isis adjacency`

Show IS-IS route database
`show isis route`

Show IS-IS link state database
`show isis database`

Show IS-IS routes that are actually installed in the RIB
`show route isis`

### BGP

Display the BGP part of the configuration
`show run router bgp`

Display a summarized version of the BGP neighbors
`show bgp ipv4 unicast summary`
`show bgp ipv6 unicast summary`

Display detailed information about a particular BGP neighbor
`show bgp neighbor <neighbor-ip>`
