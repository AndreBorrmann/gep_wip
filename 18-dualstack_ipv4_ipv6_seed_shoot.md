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

### User facing Configuration

#### IPFamilies in Shoot specification

Shoot spec should now support IPFamilies, like the [k8s service](https://kubernetes.io/docs/concepts/services-networking/dual-stack/#services) does. Order of ipFamilies should not matter. The field should be immutable. **As soon as `IPv4` and `IPv6` is set in ipFamilies, the cluster should be considered as dual stack cluster.** It should help us in improving validation and while being in operation precedures. This should default to `["IPv4"]`. 

If the Shoot is a dual stack cluster, the dual stack feature gate on all kubernetes components **must** be enabled.

Allowed values should be:
- `["IPv4"]`
- `["IPv4","IPv6"]` (dual stack)
- `["IPv6","IPv4"]` (dual stack)

```yaml
apiVersion: v1beta1
kind: Shoot
spec:
  kubernetes:
    ipFamilies:
    - IPv4
    - IPv6
```

#### CIDRs should now support comma seperated networks

Shoot networks definitions **must** have a dual stack network definitions as soon as the Shoot should be a dual stack cluster.

```yaml
apiVersion: core.gardener.cloud/v1beta1
kind: Shoot
spec:
  networking:
    pods: '100.96.0.0/11,2a05:6000:caf9::/64'
    nodes: '10.250.0.0/16,2a05:6000:caf9:0001::/64'
    services: '100.64.0.0/13,2a05:6000:caf9:0002::/64'
```

#### Node CIDR mask size for v4 and v6

If the Shoot is not a dual stack cluster `nodeCIDRMaskSize` should be used as usual. The fields `nodeCIDRMaskSizeV4` and `nodeCIDRMaskSizeV6` should be ignored.

If the Shoot is a dual stack cluster, `nodeCIDRMaskSizeV4` and `nodeCIDRMaskSizeV6` should be used and `nodeCIDRMaskSize` should be ignored. The default of the values should be the same as in the [Kube-Controller-Manager](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/). 

The default value for `nodeCIDRMaskSize` and `nodeCIDRMaskSizeV4` should be `24`. 
The default value for `nodeCIDRMaskSizeV6` should be `64`.

```yaml
apiVersion: core.gardener.cloud/v1beta1
kind: Shoot
spec:
  kubernetes:
    kubeControllerManager:
      nodeCIDRMaskSize: 24
      nodeCIDRMaskSizeV4: 24
      nodeCIDRMaskSizeV6: 64
```


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

>TODO

#### CIDR's may require to allow to use template formats

As seen in the section above it might not be possible to know the IPv6 CIDRs assigned to the Gardener VPC's while creating the `shoot` configuration as they are commisioned by the cloud provider. Therefore the specific sub-nets might not be able to be fully specified as part of this configuration. It should be allowed to use specific template format when specifying the sub net CIDRs within the shoot spec. This would allow a specific web-hook that get's called with the actual created/assigned IPv6 address ranges from the provider to complete the configuration and thus allows the creation of the sub-nets.

Example:

```yaml
  networking:
    type: calico
    pods: 100.128.0.0/11, <PROVIDER/64>:ffff:1100::/108
    nodes: 10.255.0.0/16, <PROVIDER/64>::/56
    services: 100.72.0.0/13, <PROVIDER/64>:ffff:3300::/108
```

In this example the template defines to use the IPv6 address assigned by the provider and mask 56 bits of it to define the basis of the sub-net. Than add some further IPv6 sub-address range with the corresponding mask size to calculate the final IPv6 CIDR. Given a cloud provider assigned IPv6 like `2a05:d014:cfe:1f00::` the resulting configuration would be

```yaml
  networking:
    type: calico
    pods: 100.128.0.0/11, 2a05:d014:cfe:1f00:ffff:1100::/108
    nodes: 10.255.0.0/16, 2a05:d014:cfe:1f00::/56
    services: 100.72.0.0/13, 2a05:d014:cfe:1f00:ffff:3300::/108
```

This enhancement also impacts how the provider specific networks need to be configured in the shoot spec. Example:

```yaml
  provider:
    type: aws # {aws,azure,gcp,...}
    infrastructureConfig:
      apiVersion: aws.provider.extensions.gardener.cloud/v1alpha1
      kind: InfrastructureConfig
      networks:
        vpc:
          cidrv4: 10.255.0.0/16
          cidrv6: <PROVIDER/64>
```

This configuration would trigger the usage of the provider specific web-hook to retrieve the assigned IPv6 address range. The template format specified here is used to validate with the occurences in the sub-net specifications. If they don't match - either the name or the mask - it is a configuration error.
