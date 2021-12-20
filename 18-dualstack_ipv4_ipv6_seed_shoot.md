# Enable IPv6 Support in Gardner Seed and Shoot Clusters

## Summary

The actual Gardener implementation takes the assumption that all components that does networking are using IPv4 singlestack only. This GEP requests to enhance the Gardener networking stack to add support for IPv6 single-stack and IPv6+IPv4 dual-stack.

## Motivation

As the transition from IPv4 to IPv6 is advancing, more endpoints will be offered as IPv6 only and connections via IPv4 will suffer from more workarounds like NAT. Therefore to be able to serve the requirements of today and the future, a transition between single stack IPv4 and single stack IPv6 needs to be performed. 
The easiest solution is to provide both protocol stacks - IPv4/IPv6 dual stack.

## Goals

- Gardener should be able to create IPv6 only shoot clusters.
- Gardener should be able to create IPv4+IPv6 dual-stack seed and shoot clusters. As k8s services in a dual-stack cluster are still single-stack by default, the preference for single-stack services should be configureable.
- Gardener should be able to use unique addresses as IPv6 cidr, thus NAT is not needed. In contrast to an IPv4 setup where you can use the same network cidrs for all shoots
- Gardener should be able to automatically get unique IPv6 cidrs for shoots from the infrastructure provider. This also helps in a ManagedSeedSet environment.
- Infrastructure provider AWS, GCP and OpenStack should support dual-stack behavior and limited IPv6 single-stack behavior.

## Non-Goals

- Realising an infrastructure-independet IPv4+IPv6 ingress for IPv4 only clusters. 
- Implement an IPv6 Layer for cloud providers, who do not support IPv6 in their infrastructure.
- Adding an infrastructure independent IPAM to Gardener.

## Proposal

> TODO

### Configuration

> TODO

### Impacted Components

> TODO

### General Considerations

#### IPAM Setup and Process Flow

This section describes the proposed configuration and process flow for the IP address management or more concrete how the CIDR ranges for nodes, pods and services are optained, configured and used during the cluster setup.

> The current implementations of IPv6 VPCs in AWS and GCP strictly limit the way of using IPv6 to scenarios that maintain uniqueness and route-ability of IPv6 addresses. As a consequence, they enforce that IPv6 prefixes are administered by the cloud provider and assigned from their pool of addresses or a bring-your-own-IPv6 pool. In this GEP, we reflect this situation with the situation and assume IPv6 ranges are usually assigned by/through the cloud provider.

In the shoot specification we will use the definition of the `network` section to provide the cidrs for `nodes`, `pods` and `services`. As those fields already exist they will be enhanced to accept a tuple of values. One value for IPv4 and one for IPv6. For the IPv6 cidr the special key word `AUTOv6` will be valid to be used to indicate that the respective CIDR will be available at runtime only and being configured during the cluster deployment/reconciliation.

Example:

```yaml
  networking:
    type: calico # {calico,cilium}
    pods: AUTOv6, 100.128.0.0/11, 
    nodes: AUTOv6, 10.255.0.0/16
    services: AUTOv6, 100.72.0.0/13
```

As the IPv6 deployment strategy of the cloud providers tries to assigning a single prefixes per host, we have to tolerate that the pod- and node-CIDR are overlapping/identical. 
The size of the prefix assigned per host depends on the cloud provider, but can be assumed _large enough_ to accommodate a per-node pod prefix. For example, GCP always assigns a `/96` and for AWS requires to use a `/80`.  For now, we assume the node IP shall be taken from the very first available IP address of this range and the first /100 is reserved for node purposes.

The IPAM flow can be seen like follows:

1. VPC creation

  The Gardener-Provider-Extension (GPE) will either use an existing or create a new VPC. The spec SHOULD not contain any IPv6 prefixes, but the place-holder `AUTOv6` as in the example above. The prefixes actual used are then derived from the prefixes the cloud provider assigned to the VPC. 
  
The extension SHOULD als take one of the following actions to make the prefixes available for K8S:

  - Store the actual subnet cidrs that shall be used for nodes, pods, services as part of the shoot-spec status
  - Store the actual subnet cidrs that shall be used within the gardener networking resource available in the shoot namespace.

2. Node creation

  The Gardener-Machine-Controler-Extension (GMCE) will create the host/Node and retrieve the actual assigned IPv6 address/prefix from the cloud provider. Based on a built-in/configureable template, the GMCE will then assign a per-node pod prefix, e.g., `<nodePrefix>::f000:0/100`, and update the K8s node resource with the respective `podCIDR` spec.

3. CNI Config

  The CNI (eg. calico) configuration uses the pod-cidrs maintained in the K8s node objekt to configure the respective networks. No adjustments are expected to be required there as those already support IPv4 as well as IPv6 in single and dual stack variations.

4. Service CIDR

Service CIDRs might be required to be disjunct from nodes and pods CIDRs.  We currently envision two deployment strategies:

- If the service addresses need to be globally unique, e.g., as they are intended to be used in a federated service mesh, these could be generated from subnet/VPC reservation (AWS) or taken from a dummy node's prefix (GCP).
- Otherwise, we can use a to-be-standardised well-known not globally unique prefix from the Global Unicast address range. 

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


