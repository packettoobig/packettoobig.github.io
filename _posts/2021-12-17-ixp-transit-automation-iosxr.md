---
title: IXP and Transit automation using the IOS-XR platform
slug: ixp-transit-automation-iosxr
redirect_from: 'ixp-transit-automation-iosxr/'
date_published: 2021-12-21 12:00:00
date_updated: 2021-10-21 12:00:00
tags: BGP, Cisco
excerpt: Peering and transit made easy
---

This post might be consequent, so I will update it if there are mistakes or typos.  
I might also improve the automation in the future.

## Introduction

I've been quite busy recently with a backbone project.

We've been running our peering and transit on IOS routers for quite some time, but we also had secondary NCS routers.

Now, we're moving some IXP and transit links to these routers, and we're planning on replacing the rest in the next two to three years.

## Thank you notes

Before explaining everything, let me thank [Renato Almeida de Oliveira](https://github.com/renatoalmeidaoliveira) for his [netero](https://github.com/renatoalmeidaoliveira/netero) code, because that helped me a lot in this project.

I also share my updated code and a sample role in return, so stay tuned for that.

## What does one need to build IXP and transit automation

The first question we asked ourselves was : "what data do we need", the second one being "how do we make it as dynamic as possibe".

For IXP peerings, it seemed quite obvious :

- PeeringDB must be the source of truth for :

  - Maximum prefix
  
  - Peer IP address
  
- The IRR (Internet Routing Registries) must be the source of truth for prefix-set filtering

Transit is a different story, but generally speaking, we tend to trust them more (because we have to).

## Full replace or go home

We've been running "full replace" configs on NX-OS and were quite happy about the result, so, we decided on doing the same thing on IOS XR.

However, in this blogpost I will only share a partial config, which is the BGP and route-policy section.

## Generic settings

### BGP related settings

```yaml
local_as: 65000

router_id: 192.0.2.1

transit_local_pref: 100
ixp_local_pref: 300

# In %
maxprefix_warning_threshold: "75"
# In minutes
maxprefix_retry: "60"
```

## IXP settings

### Who do we want to peer with

We want something as easy as possible as a starter, so let's just create a group_var list of which networks we want to peer with.

```yaml
desirable_ixp_neighbors:
  - '15169' #Google
  - '714' #Apple
  - '32934' #Facebook
  - '16509' #Amazon
  - '8075' #Microsoft
  - '13335' #Cloudflare
  - '20940' #Akamai
  - '2906' #Netflix
  - '6939' #Hurricane Electric
```

It's that simple.  
And yes, it makes things more difficult in case we want to set a password for example.
But in the future we can convert this list to a dict, if required.

### Additional IXP settings

We need to know to which IXP each router is connected.  
We could do that by looking at the interfaces address and checking peeringDB, but we decided that it was really simple to just put that in the host_vars instead.

Here is an example :

```yaml
ixp:
  - name: 'DE-CIX Frankfurt'
    shortname: 'decix_fra'
    peeringdb_id: '31'
    rs_asn: 6695
    rs_addresses:
      - '80.81.192.157'
      - '80.81.193.157'
      - '2001:7f8::1a27:5051:c09d'
      - '2001:7f8::1a27:5051:c19d'
```

### Additional Transit settings

Transit will never be dynamic, or using any public database.  
So, we will just use host_vars like so:

```yaml
transits:
  - name: "Twelve99/TSIC"
    shortname: tsic
    asn: 1299
    addresses:
      - '192.0.2.1'
      - '2001:db8:0:beef::1'

  - name: "Lumen/CenturyLink/Level3"
    shortname: level3
    asn: 3356
    addresses:
      - '198.51.100.1'
      - '2001:db8:1:face::1'
```

## Putting the PeeringDB IXP part together

Now we need something that ties it all together, something that will get the information from PeeringDB, and check if this particular peer is on the same IXP as us.

This is where our [custom ansible modules](https://github.com/packettoobig/ansible-sample-transit-ix-iosxr/blob/master/roles/discover/library/peeringdb_getasn.py) comes into play.  

The module can be used like so :

```yaml
  - name: Get peeringDB info for each desirable ASN and IXP
    peeringdb_getasn:
      asn: "{{ item.0 }}"
      ix_id: "{{ item.1.peeringdb_id | int }}"
      api_key: "{{ peeringdb_api_key }}"
    delegate_to: localhost
    async: 180
    poll: 0
    loop: "{{ desirable_ixp_neighbors | product(ixp) }}"
    loop_control:
      label: "AS{{ item.0 }} on {{ item.1.name }}"
    register: peeringdb_task
    when: ixp is defined and ixp is iterable
```

Did you notice the "async" mention ?
This is a way for us to make the playbook run faster with "fire and forget", and only getting the results back with the next task:

The API key is not strictly necessary for the information we're grabbing, but peeringDB is supposedly applying a stronger rate-limiting on unauthenticated users, even if I did not actually test this.

```yaml
  - name: Wait for async peeringDB tasks completion and register results
    async_status:
      jid: "{{ item.ansible_job_id }}"
    delegate_to: localhost
    retries: 60
    delay: 3
    loop: "{{ peeringdb_task.results }}"
    loop_control:
      label: "AS{{ item.item.0 }} on {{ item.item.1.name }}"
    register: peeringdb
    when: peeringdb_task is defined and ixp is defined and ixp is iterable
    until: peeringdb.finished
```

Then, we filter the data, and create both a list of ASN, and a dictionnary with all the required config information.

```yaml
  - name: Filter peeringdb results into local_ixp_neighbors dict and local_ixp_asn list
    set_fact:
      local_ixp_neighbors: "{{ local_ixp_neighbors|default([]) + [ helper_dict ] }}"
      local_ixp_asn: "{{ (local_ixp_asn|default([]) + [ helper_dict.asn ]) | unique }}"
    delegate_to: localhost
    loop: "{{ peeringdb.results }}"
    loop_control:
      label: "AS{{ item.item.item.0 }} on {{ item.item.item.1.name }}"
    when: peeringdb is defined and item.message.interfaces is defined # Only neighbors with at least one interface connected to the IXP
    vars:
      helper_dict:
        asn: "{{ item.message.asn }}"
        maxprefix_v4: "{{ item.message.info_prefixes4|default(omit) }}"
        maxprefix_v6: "{{ item.message.info_prefixes6|default(omit) }}"
        interfaces: "{{ item.message.interfaces|default(omit) }}"
        as_set: "{{ item.message.irr_as_set|default(omit) }}"
        contact: "{{ item.message.poc_set|default(omit) }}"
```

## Filtering with the IRR

Traditionally, people use bgpq3 or bgpq4 to create prefix-lists (orin our case, prefix-sets).

That is exactly what we will do with the IRR ansible module.

This is where our [custom ansible modules](https://github.com/packettoobig/ansible-sample-transit-ix-iosxr/blob/master/roles/discover/library/irr_prefix.py) comes into play.

The module can be used like so :

```yaml
  - name: Launch one async bgpq3/4 task per ASN per AF
    irr_prefix:
      IPv: "{{ item.1 }}"
      ASN: "AS{{ item.0 }}"
      irrd_host: 'rr.ntt.net'
    delegate_to: localhost
    async: 180
    poll: 0
    loop: "{{ local_ixp_asn | product(['4', '6']) }}"
    register: irr_task
    when: local_ixp_asn is defined and local_ixp_asn is iterable
```

As with the peeringdb module, we run it asynchronously, and get the information with the next task.

```yaml
  - name: Wait for async irr tasks completion and register results
    async_status:
      jid: "{{ item.ansible_job_id }}"
    delegate_to: localhost
    retries: 60
    delay: 3
    loop: "{{ irr_task.results }}"
    loop_control:
      label: "{{ item.item }}"
    register: irr
    when: irr_task is defined and local_ixp_asn is defined and local_ixp_asn is iterable
    until: irr.finished
```

## What about the transits ?

Well, there is not really much we can do about them in the playbook itself.
But we will use the information we have in the template

## Templating

The most time consuming part is to figure out what you want to accept, and what you want to reject.

Here is the template link that puts all of it together.

INSERT GITHUB LINK TO TEMPLATE.

### Section 1 : Dynamic prefix-sets

I won't put the extracts in the blog post (or it might end up being too long).

But you can see that we create two `PS-IRR-ASXXXX` prefix-sets per neighbor, one v4 and one v6.  
They are based on the irr_prefix.py module result.

### Section 2 : Static prefix-sets

Here we have V4 bogons and V6 bogons.

### Section 3 : as-path-sets

We only one as-path-set, and it is for the bogon AS.

### Section 4 : route-policies

#### 4.1 Static route-policies

The route policies are interesting.  
We gather a lot of bogon, and undesirable prefixes and as-path, and mix all of them into `RP-STD-EBGP-FILTER`, one v4, one v6.

It makes up the bulk of our filtering.

#### 4.2 "IN" IXP policies

Then, if the prefixes are accepted, we need to act on them.
We like to tag all prefixes with the neighbor ASN, to more easily identify the route-server prefixes, and the direct peer prefixes.

We use a large community for that :

```yaml
received_from_lcomm: "{{ local_as }}:2"
```

In our example case, it gives `65000:2:1299` for a prefix received from TSIC for example.

We also apply a higher local-preference to prefixes received from an IXP (because it's cheaper).  
We only do that is the as-path length is lower than 2, because we want our neighbors to be able to prepend in case of issue on their IXP router.

For this reason, we might use MED in the future instead, and have the same local-pref for every route.

#### 4.3 "OUT" IXP policies

This one is easy, we only announce if it matches our internal prefix-set, and if it is tagged with our internal "announce" community.

#### 4.4 "IN" Transit policy

This one is very simple, we simply apply our standard `RP-STD-EBGP-FILTER` and tag it with the `received_from_lcomm` large community, and with a default transit local pref.

#### 4.5 "OUT" Transit policy

We do some traffic engineering for the outbound announces, so depending on the community, we either do not announce the prefix, or prepend it a couple of times.

### Section 5 : BGP neighbors

Quite easy as well, we just create one neighbor-group per AS per address-family, and apply the route-policy based on the neighbor type (rs, peer, or transit).

The only specific thing is, sometimes, one router might be on two IXPs at once, and we do not want to have duplicate neighbor-group for that, so we just make sure that we keep track of which ones were created with these lines

```yaml
    {%- set seen_asns = [] %}
    {% for neighbor in local_ixp_neighbors %}
      {% if neighbor.asn not in seen_asns|unique %}
# And later in the file
{{- seen_asns.append(neighbor.asn) }}
```

## Running the playbook

Here is how it looks like

```shell
$ ansible-playbook pb-iosxr.yml -i inventory/inventory.ini 

PLAY [Playbook for NCS configurations] ******************************************************************************************************************************************************************

TASK [discover : Get Cymru fullbogons v4] ***************************************************************************************************************************************************************
ok: [router01]

TASK [discover : Display bogon values] ******************************************************************************************************************************************************************
ok: [router01] => {
    "msg": [
        "1444 ipv4 fullbogons found"
    ]
}

TASK [discover : Get peeringDB info for each desirable ASN and IXP] *************************************************************************************************************************************
changed: [router01] => (item=AS15169 on DE-CIX Frankfurt)
changed: [router01] => (item=AS714 on DE-CIX Frankfurt)
changed: [router01] => (item=AS32934 on DE-CIX Frankfurt)
changed: [router01] => (item=AS16509 on DE-CIX Frankfurt)
changed: [router01] => (item=AS8075 on DE-CIX Frankfurt)
changed: [router01] => (item=AS13335 on DE-CIX Frankfurt)
changed: [router01] => (item=AS20940 on DE-CIX Frankfurt)
changed: [router01] => (item=AS2906 on DE-CIX Frankfurt)
changed: [router01] => (item=AS6939 on DE-CIX Frankfurt)

TASK [discover : Wait for async peeringDB tasks completion and register results] ************************************************************************************************************************
FAILED - RETRYING: Wait for async peeringDB tasks completion and register results (60 retries left).
FAILED - RETRYING: Wait for async peeringDB tasks completion and register results (59 retries left).
ok: [router01] => (item=AS15169 on DE-CIX Frankfurt)
ok: [router01] => (item=AS714 on DE-CIX Frankfurt)
ok: [router01] => (item=AS32934 on DE-CIX Frankfurt)
ok: [router01] => (item=AS16509 on DE-CIX Frankfurt)
ok: [router01] => (item=AS8075 on DE-CIX Frankfurt)
ok: [router01] => (item=AS13335 on DE-CIX Frankfurt)
ok: [router01] => (item=AS20940 on DE-CIX Frankfurt)
ok: [router01] => (item=AS2906 on DE-CIX Frankfurt)
ok: [router01] => (item=AS6939 on DE-CIX Frankfurt)

TASK [discover : Filter peeringdb results into local_ixp_neighbors dict and local_ixp_asn list] *********************************************************************************************************
ok: [router01] => (item=AS15169 on DE-CIX Frankfurt)
ok: [router01] => (item=AS714 on DE-CIX Frankfurt)
ok: [router01] => (item=AS32934 on DE-CIX Frankfurt)
ok: [router01] => (item=AS16509 on DE-CIX Frankfurt)
ok: [router01] => (item=AS8075 on DE-CIX Frankfurt)
ok: [router01] => (item=AS13335 on DE-CIX Frankfurt)
ok: [router01] => (item=AS20940 on DE-CIX Frankfurt)
ok: [router01] => (item=AS2906 on DE-CIX Frankfurt)
ok: [router01] => (item=AS6939 on DE-CIX Frankfurt)

TASK [discover : Print the available AS lists] **********************************************************************************************************************************************************
ok: [router01] => {
    "msg": [
        "desired peer AS:         ['15169', '714', '32934', '16509', '8075', '13335', '20940', '2906', '6939']",
        "AS available locally:    ['15169', '714', '32934', '16509', '8075', '13335', '20940', '2906', '6939']",
        "AS unavailable locally:  []"
    ]
}

TASK [discover : Launch one async bgpq3/4 task per ASN per AF] ******************************************************************************************************************************************
changed: [router01] => (item=['15169', '4'])
changed: [router01] => (item=['15169', '6'])
changed: [router01] => (item=['714', '4'])
changed: [router01] => (item=['714', '6'])
changed: [router01] => (item=['32934', '4'])
changed: [router01] => (item=['32934', '6'])
changed: [router01] => (item=['16509', '4'])
changed: [router01] => (item=['16509', '6'])
changed: [router01] => (item=['8075', '4'])
changed: [router01] => (item=['8075', '6'])
changed: [router01] => (item=['13335', '4'])
changed: [router01] => (item=['13335', '6'])
changed: [router01] => (item=['20940', '4'])
changed: [router01] => (item=['20940', '6'])
changed: [router01] => (item=['2906', '4'])
changed: [router01] => (item=['2906', '6'])
changed: [router01] => (item=['6939', '4'])
changed: [router01] => (item=['6939', '6'])

TASK [discover : Wait for async irr tasks completion and register results] ******************************************************************************************************************************
changed: [router01] => (item=['15169', '4'])
changed: [router01] => (item=['15169', '6'])
changed: [router01] => (item=['714', '4'])
changed: [router01] => (item=['714', '6'])
changed: [router01] => (item=['32934', '4'])
changed: [router01] => (item=['32934', '6'])
changed: [router01] => (item=['16509', '4'])
changed: [router01] => (item=['16509', '6'])
changed: [router01] => (item=['8075', '4'])
changed: [router01] => (item=['8075', '6'])
changed: [router01] => (item=['13335', '4'])
changed: [router01] => (item=['13335', '6'])
changed: [router01] => (item=['20940', '4'])
changed: [router01] => (item=['20940', '6'])
changed: [router01] => (item=['2906', '4'])
changed: [router01] => (item=['2906', '6'])
changed: [router01] => (item=['6939', '4'])
changed: [router01] => (item=['6939', '6'])

TASK [discover : Generate local AS65000 bgpq3/4 results] ************************************************************************************************************************************************
changed: [router01] => (item=4)
changed: [router01] => (item=6)

TASK [network_config : Generate config file] ************************************************************************************************************************************************************
ok: [router01]

PLAY RECAP **********************************************************************************************************************************************************************************************
router01                   : ok=10   changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

## Config result

You can inspect an example config result [here](https://github.com/packettoobig/ansible-sample-transit-ix-iosxr/blob/master/configurations/router01).

Obviously *AS65000* has no IRR entries, so the policy is not created, but [feel free to replace](https://github.com/packettoobig/ansible-sample-transit-ix-iosxr/blob/master/inventory/group_vars/iosxr.yml) `local_as` with your own !

## Conclusion

The full code is available [in my github repo](https://github.com/packettoobig/ansible-sample-transit-ix-iosxr), clone it and try it out :)

If you spot any bug, or any improvement, don't hesitate to fix it and open a MR.
