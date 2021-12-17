# Enable IPv4 & IPv6 Dualstack Support in Gardner Seed and Shoot

## Summary

The actual Gardener implementation takes the assumption that all components that does networking are using IPv4 singlestack only. This GEP requests to enhance the Gardener networking stack to add support for IPv6 in a Dualstack or Singlestack mode.

## Motivation

As the transition from IPv4 to IPv6 is advancing, more endpoints will be offered as IPv6 only and connections via IPv4 will suffer from more workarounds like NAT. Therefore to be able to serve the requirements of today and the future, a transition between single stack IPv4 and single stack IPv6 needs to be performed. 
The easiest solution is to provide both protocol stacks - IPv4/IPv6 dual stack.

## Goals

- Gardener should be able to create shoots with dual-stack enabled
- Gardener should be able to use unique addresses as IPv6 cidr, thus NAT is not needed. In contrast to an IPv4 setup where you can use the same network cidrs for all shoots
- Gardener should be able to automatically get unique IPv6 cidrs for Shoots, this also helps in a ManagedSeedSet environment
- Infrastructure provider AWS and OpenStack should support dual-stack behavior

## Non-Goals

- IPv6 single stack support
- Implement an IPv6 Layer for cloud providers, who do not support IPv6 in their infrastructure

## Proposal

> TODO

### Configuration

> TODO

### Impacted Components

> TODO

### General Considerations

> TODO

### Cloud Provider / Hyperscaler Constraints

The different Cloud Provider do handle the IPv6 networking configurations in different ways. Those specific constraints required to be addressed within the specific Gardener Provider Extensions:

**AWS :**

AWS provides two options to choose from:

- retrieve the IPv6 addresses and subnets from an AWS owned address pool
- "bring your own IPv6" address and subnet to be used for the VPC

**GCP :**

GCP only allows to choose IPv6 addresses and subnets from an address pool owned by GCP.
The subnets created for an individual node does have a */96* size.

**Azure :**

Azure does only allow to use "your own IPv6" address and subnet. For external communications the addresses will always be translated using a NAT gateway. Thus Azure proposes to only use local/private IPv6 addresses/cidrs. 

**OpenStack :**

OpenStack provides the possibility to create subnet pools. If a subnet is allocated from a subnet pool, OpenStack guarantees that the subnet does not collide with any subnet derriving from the same subnet pool. 
A Subnet and a subnet pool can either be IPv4 or IPv6. Although in this proposal we can ignore IPv4 subnet pools completely.

OpenStack router can be attached to a subnet. Therefor we need a router for each subnet.
Routers propagate the networks attached to its upstream network.

To make sure the router propagates the Pod network to the upstream network, the Pod network must be part of the IPv6 Subnet attached to the Nodes. 

```
                                    ┌────────────────────────────────────┐    ┌────────────────────────────────────┐
                                    │                                    │    │                                    │
                                    │            Upstream v4             │    │             Upstream v6            │
                                    │                                    │    │                                    │
                                    └────────────────────────────────────┘    └────────────────────────────────────┘
                                                      ▲                                           ▲
                                                      │                                           │
                                               ┌──────┴───────┐                           ┌───────┴──────┐
                                               │              │                           │              │  Routing Entries:
                                               │  Router      │                           │  Router      │  2001:DB8:0:0:4000:0:0:0/112 via 2001:DB8::a1
                                               │              │                           │              │  2001:DB8:0:0:4000:0:1:0/112 via 2001:DB8::b4
                                               └──────────────┘                           └──────────────┘
                                                      ▲                                           ▲
                                                      │                                           │
         CIDR: 100.0.0.0/24             ┌─────────────┴────────────┐               ┌──────────────┴──────────┐ CIDR: 2001:DB8::/64
         Allocation Pool: 100.0.0.0/24  │     Subnet v4            │               │       Subnet v6         │ Allocation Pool: 2001:DB8::/66
                                    ┌───┴──────────────────────────┴───────────────┴─────────────────────────┴──────┐
                                    │                                                                               │
                                    │                                                                               │
                                    │                               Network                                         │
                                    │                                                                               │
                                    │                                                                               │
                                    └───────────────────────────────────────────────────────────────────────────────┘
                                                     ▲                                          ▲
                                                     │                                          │
┌────────────────────────────────────────────────────┼──────────────────────────────────────────┼────────────────────────────────────────────────────┐
│                                                    │                  Shoot                   │                                                    │
│                                                    │                                          │                                                    │
│                                               ┌────┴─────┐                              ┌─────┴────┐                                               │
│                                               │  Port    │                              │  Port    │                                               │
│                                             ┌─┴──────────┴─┐                          ┌─┴──────────┴─┐                                             │
│              IPv4: 100.0.0.10               │              │                          │              │              IPv4: 100.0.0.15               │
│              IPv6: 2001:DB8::a1/66          │   Node       │                          │   Node       │              IPv6: 2001:DB8::b4/66          │
│ Assigned Pod CIDR: 2001:DB8::4000:0:0:0/112 │              │                          │              │ Assigned Pod CIDR: 2001:DB8::4000:0:1:0/112 │
│                                             └──────────────┘                          └──────────────┘                                             │
│                                                                                                                                                    │
│                                                                                                                                                    │
│                                                                                                                                                    │
│                                                                                                                                                    │
│ Shoot Properties                                                                                                                                   │
│ Node CIDR: 100.0.0.0/24,2001:DB8::/66                                                                                                              │
│ Pod CIDR: 10.0.0.0/16,2001:DB8::4000:0:0:0/66                                                                                                      │
│ Svc CIDR: 10.0.0.0/16,2001:DB8::8000:0:0:0/66                                                                                                      │
│                                                                                                                                                    │
└────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

Dual-Stack Solution:
- IPv4 and IPv6 subnets **MUST** be attached to the same Network. (single home)
- IPv6 subnet **CAN** be allocated from a subnet pool
- The DHCPv6 of the subnet **MUST** serve apart 
- A router **MUST** be attached to the IPv6 subnet
- Pod subnet **MUST** be part of the IPv6 subnet, so that packages that will be routed to a upstream network can find a way back to the router
- There **MUST** be a mechanism in place to tell the router pod cidrs per node, so packages that were routed to a upstream network can find their way back to the pod
- Service subnet **CAN** be part of the IPv6 subnet, so that service IPs can be routable in the upstream network
- There **CAN** be a mechanism in place to tell the router, where to find destinations of service ip addresses, so that packages sent to the service IP can reach a node serving the message. 


