---
title: Junos quick troubleshooting command reference
slug: junos-tshoot-command-ref
redirect_from : 
  - '/junos-tshoot-command-ref'
  - '/junos-tshoot-command-ref/'
date_published: 2019-10-10T07:18:12.000Z
date_updated: 2021-04-20T07:27:09.000Z
tags: Juniper
excerpt: A nice and handy Junos command reference
---

Hey, because it is always handy to have a quick command reference, here are a few useful ones that I sometimes need to find quickly.
I will try to keep it up-to-date.

## Interfaces

Display the interface part of the configuration
`show configuration interfaces`

Quickly check L1 and L2 status (BPDUs, tx and rx errors)
`show interfaces <interface-name> media`

SFP/Transceiver/Fibre checking
`show interfaces diagnostics optics <interface-name>`

List all interfaces with their IP
`show interfaces terse`

Find out which interface is part of which VRF
`show interfaces routing-instance all terse`

## LACP

Show the status of all LACP interfaces:
`show lacp interfaces`
`show lacp statistics interfaces`

## ARP and NDP

Display the ARP table (ipv4)
`show arp`

Display the ND table (ipv6)
`show ipv6 neighbors`

## Hardware troubleshooting

Temperature and fans
`show chassis environment`

Serial numbers and PID
`show chassis hardware`

## System troubleshooting

Display the system part of the configuration
`show configuration system`

Uptime and NTP
`show system uptime`

NTP servers
`show ntp associations`

Check the proper DNS resolution of a FQDN
`show host example.com`

Currently logged-in users
`show system users`

Last configuration changes with time and date
`show system commit`

System and modules version
`show version`

## Routing protocols troubleshooting

### IS-IS

Show the IS-IS part of the configuration
`show configuration protocols isis`

Show on which interfaces is IS-IS running
`show isis interface`

Show on which interface IS-IS actually established an adjacency
`show isis adjacency`

Show IS-IS route database
`show isis route`

Show IS-IS link state database
`show isis database`

Show IS-IS routes that are actually installed in the RIB
`show route protocol isis`

### BGP

Display the BGP part of the configuration
`show configuration protocols bgp`

Display a summarized version of the BGP neighbors (I highly prefer the Cisco one)
`show bgp summary`

Display detailed information about a particular BGP neighbor
`show bgp neighbor <neighbor-ip>`
