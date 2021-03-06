---

title: Designing Virtual networks with NAT gateway resources
titleSuffix: Azure Virtual Network NAT
description: Learn how to design virtual networks with NAT gateway resources.
services: virtual-network
documentationcenter: na
author: asudbring
manager: KumudD
ms.service: virtual-network
Customer intent: As an IT administrator, I want to learn more about how to design virtual networks with NAT gateway resources.
ms.devlang: na
ms.topic: overview
ms.tgt_pltfrm: na
ms.workload: infrastructure-services
ms.date: 02/18/2020
ms.author: allensu
---

# Designing virtual networks with NAT gateway resources (Preview)

NAT gateway resources are part of [Virtual Network NAT](nat-overview.md) and provide outbound Internet connectivity for one or more subnets of a virtual network. The subnet of the virtual network states which NAT gateway will be used. NAT provides source network address translation (SNAT) for a subnet.  NAT gateway resources specify which static IP addresses virtual machines use when creating outbound flows. Static IP addresses come from public IP address resources, public IP prefix resources, or both. A NAT gateway resource can use up to 16 static IP addresses from either.


<p align="center">
  <img src="media/nat-overview/flow-direction1.svg" width="256" title="Virtual Network NAT for outbound to Internet">
</p>

*Figure: Virtual Network NAT for outbound to Internet*


>[!NOTE] 
>Virtual Network NAT is available as public preview at this time. Currently it's only available in a limited set of [regions](nat-overview.md#region-availability). This preview is provided without a service level agreement and isn't recommended for production workloads. Certain features may not be supported or may have constrained capabilities. See the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.comsupport/legal/preview-supplemental-terms) for details.

> [!IMPORTANT]
> After Virtual Network NAT [preview is enabled](./nat-overview.md#enable-preview) on your subscription, use https://aka.ms/natportal to access the portal.

## How to deploy NAT

Configuring and using NAT gateway is intentionally made simple:  

NAT gateway resource:
- Create regional or zonal (zone-isolated) NAT gateway resource,
- Assign IP addresses,
- Modify idle timeout (optional).

Virtual network:
- Configure virtual network subnet to use a NAT gateway.

User-defined routes aren't necessary.

## Resource

The resource is designed to be simple as you can see from the following Azure Resource Manager example in a template-like format.  This template-like format is shown here to illustrate the concepts and structure.  Modify the example for your needs.  This document isn't intended as a tutorial.

The following diagram shows the writeable references between the different Azure Resource Manager resources.  The arrow indicates the direction of the reference, originating from where it's writeable. Review 

<p align="center">
  <img src="media/nat-overview/flow-map.svg" width="256" title="Virtual Network NAT object model">
</p>

*Figure: Virtual Network NAT object model*

NAT is recommended for most workloads unless you have a specific dependency on [pool-based Load Balancer outbound connectivity](../load-balancer/load-balancer-outbound-connections.md).  

You can migrate from standard load balancer scenarios, including [outbound rules](../load-balancer/load-balancer-outbound-rules-overview.md), to NAT gateway. To migrate, move the public ip and public ip prefix resources from load balancer frontends to NAT gateway. New IP addresses for NAT gateway aren't required. Standard public IP and prefix can be reused as long as the total doesn't exceed 16 IP addresses. Plan for migration with service interruption in mind during the transition.  You can minimize the interruption by automating the process. Test the migration in a staging environment first.  During the transition, inbound originated flows aren't affected.

The following example would create a NAT gateway resource called _myNATGateway_ is created in region _East US 2, AZ 1_ with a _4-minutes_ idle timeout. The outbound IP addresses provided are:
- A set of public IP address resources _myIP1_ and _myIP2_ and 
- A set of public IP prefix resources _myPrefix1_ and _myPrefix2_. 

The total number of IP addresses provided by all four IP address resources can't exceed 16 IP addresses total. Any number of IP addresses between 1 and 16 is allowed.

```json
{
"name": "myNATGateway",
   "type": "Microsoft.Network/natGateways",
   "apiVersion": "2018-11-01",
   "location": "East US 2",
   "sku": { "name": "Standard" },
   "zones": [ "1" ],
   "properties": {
      "idleTimeoutInMinutes": 4, 
      "publicIPPrefixes": [
         {
            "id": "ref to myPrefix1"
         },
         {
            "id": "ref to myPrefix2"
         }
      ],
      "publicIPAddresses": [
         {
            "id": "ref to myIP1"
         },
         {
            "id": "ref to myIP2"
         }
      ]
   }
}
```

When the NAT gateway resource has been created, it can be used on one or more subnets of a virtual network. Specify which subnets use this NAT gateway resource. A NAT gateway isn't able to span more than one virtual network. It isn't required to assign the same NAT gateway to all subnets of a virtual network. Individual subnets can be configured with different NAT gateway resources.

Scenarios that don't use availability zones will be regional (no zone specified). If you're using availability zones, you can specify a zone to isolate NAT to a specific zone. Zone-redundancy isn't supported. Review NAT [availability zones](#availability-zones).


```json
{
   "name": "myVNet",
   "apiVersion": "2018-11-01",
   "type": "Microsoft.Network/virtualNetworks",
   "location": "myRegion", 
   "properties": {
      "addressSpace": {
          "addressPrefixes": [
           "192.168.0.0/16"
          ]
      },
      "subnets": [
         {
            "name": "mySubnet1",
            "properties": {
               "addressPrefix": “192.168.0.0/24",
               "natGateway": {
                  "id": "ref to myNATGateway"
               }
            }
         } 
      ]
   }
}
```
NAT gateways are defined with a property on a subnet within a virtual network. Flows created by virtual machines on subnet _mySubnet1_ of virtual network _myVNet_ will use the NAT gateway. All outbound connectivity will use the IP addresses associated with _myNatGateway_ as the source IP address.

## Coexistence of inbound and outbound

NAT gateway is compatible with:

 - Standard load balancer
 - Standard public IP
 - Standard public IP prefix

When developing a new deployment, start with standard SKUs.

<p align="center">
  <img src="media/nat-overview/flow-direction1.svg" width="256" title="Virtual Network NAT for outbound to Internet">
</p>

*Figure: Virtual Network NAT for outbound to Internet*

The Internet outbound only scenario provided by NAT gateway can be expanded with inbound from Internet functionality. Each resource is aware of the direction in which a flow is originated. On a subnet with a NAT gateway, all outbound to Internet scenarios are superseded by the NAT gateway. Inbound from Internet scenarios are provided by the respective resource.

### NAT and VM with instance-level Public IP

<p align="center">
  <img src="media/nat-overview/flow-direction2.svg" width="300" title="Virtual Network NAT and VM with instance-level Public IP">
</p>

*Figure: Virtual Network NAT and VM with instance-level Public IP*

| Direction | Resource |
|:---:|:---:|
| Inbound | VM with instance-level Public IP |
| Outbound | NAT gateway |

VM will use NAT gateway for outbound.  Inbound originated isn't affected.

### NAT and VM with public Load Balancer

<p align="center">
  <img src="media/nat-overview/flow-direction3.svg" width="350" title="Virtual Network NAT and VM with public Load Balancer">
</p>

*Figure: Virtual Network NAT and VM with public Load Balancer*

| Direction | Resource |
|:---:|:---:|
| Inbound | public Load Balancer |
| Outbound | NAT gateway |

Any outbound configuration from a load-balancing rule or outbound rules is superseded by NAT gateway.  Inbound originated isn't affected.

### NAT and VM with instance-level public IP and public Load Balancer

<p align="center">
  <img src="media/nat-overview/flow-direction4.svg" width="425" title="Virtual Network NAT and VM with instance-level public IP and public Load Balancer">
</p>

*Figure: Virtual Network NAT and VM with instance-level public IP and public Load Balancer*

| Direction | Resource |
|:---:|:---:|
| Inbound | VM with instance-level public IP and public Load Balancer |
| Outbound | NAT gateway |

Any outbound configuration from a load-balancing rule or outbound rules is superseded by NAT gateway.  The VM will also use NAT gateway for outbound.  Inbound originated isn't affected.

## Managing Basic resources

Standard load balancer, public IP, and public IP prefix are compatible with NAT gateway. NAT gateways operate in the scope of a subnet. The basic SKU of these services must be deployed on a subnet without a NAT gateway. This separation allows both SKU variants to coexist in the same virtual network.

NAT gateways take precedence over outbound scenarios of the subnet. Basic load balancer or public IP (and any managed service built with them) is unable to be adjusted with the correct translations. NAT gateway takes control over outbound to Internet traffic on a subnet. Inbound traffic to basic load balancer and public ip is unavailable. Inbound traffic to a basic load balancer and, or a public ip configured on a VM won't be available.

## Availability Zones

Even without availability zones, NAT is resilient and can survive multiple infrastructure component failures. When availability zones are part of your scenario, you should configure NAT for a specific zone.  The control plane operations and data plane are constrained to the specified zone. Failure in a zone other than where your scenario exists is expected to be without impact to NAT. Outbound traffic from virtual machines in the same zone will fail because of zone isolation.

<p align="center">
  <img src="media/nat-overview/az-directions.svg" width="425" title="Virtual Network NAT with availability zones">
</p>

*Figure: Virtual Network NAT with availability zones*

A zone-isolated NAT gateway requires IP addresses to match the zone of the NAT gateway. NAT gateway resources with IP addresses from a different zone or without a zone are unsupported.

Virtual networks and subnets are regional and not zonal aligned. A VM must be in the same zone as NAT gateway for a zonal promise of outbound connections. Zone isolation is created by creating a zonal "stack" per availability zone. A zonal promise won't exist when crossing zones of a zonal NAT gateway or using a regional NAT gateway with zonal VMs.

When you deploy virtual machine scale sets to use with NAT, you deploy a zonal scale set on its own subnet and attach the matching zone NAT gateway to that subnet. If you use zone-spanning scale sets (a scale set in two or more zones), NAT won't provide a zonal promise.  NAT doesn't support zone-redundancy.  Only regional or zone-isolation is supported.

<p align="center">
  <img src="media/nat-overview/az-directions2.svg" width="425" title="zone-spanning Virtual Network NAT">
</p>

*Figure: Zone-spanning Virtual Network NAT*

The zones property isn't mutable.  Redeploy NAT gateway resource with the intended regional or zone preference.

>[!NOTE] 
>IP addresses by themselves aren't zone-redundant if no zone is specified.  The frontend of a [Standard Load Balancer is zone-redundant](../load-balancer/load-balancer-standard-availability-zones.md#frontend) if an IP address isn't created in a specific zone.  This doesn't apply to NAT.  Only regional or zone-isolation is supported.

## Source Network Address Translation

Source network address translation (SNAT) rewrites the source of a flow to originate from a different IP address.  NAT gateway resources use a variant of SNAT commonly referred to port address translation (PAT). PAT rewrites the source address and source port. With SNAT, there's no fixed relationship between the number of private addresses and their translated public addresses.  

### Fundamentals

Let's look at an example of four flows to explain the basic concept.  The NAT gateway is using public IP address resource 65.52.0.2.

| Flow | Source tuple | Destination tuple |
|:---:|:---:|:---:|
| 1 | 192.168.0.16:4283 | 65.52.0.1:80 |
| 2 | 192.168.0.16:4284 | 65.52.0.1:80 |
| 3 | 192.168.0.17.5768 | 65.52.0.1:80 |
| 4 | 192.168.0.16:4285 | 65.52.0.2:80 |

These flows might look like this after PAT has taken place:

| Flow | Source tuple | SNAT'ed source tuple | Destination tuple | 
|:---:|:---:|:---:|:---:|
| 1 | 192.168.0.16:4283 | 65.52.0.2:234 | 65.52.0.1:80 |
| 2 | 192.168.0.16:4284 | 65.52.0.2:235 | 65.52.0.1:80 |
| 3 | 192.168.0.17.5768 | 65.52.0.2:236 | 65.52.0.1:80 |
| 4 | 192.168.0.16:4285 | 65.52.0.2:237 | 65.52.0.2:80 |

The destination will see the source of the flow as 65.52.0.2 (SNAT source tuple) with the assigned port shown.  PAT as shown in the preceding table is also called port masquerading SNAT.  Multiple private sources are masqueraded behind an IP and port.

Don't take a dependency on the specific way source ports are assigned.  The preceding is an illustration of the fundamental concept only.

SNAT provided by NAT is different from [Load Balancer](../load-balancer/load-balancer-outbound-connections.md) in several aspects.

### On-demand

NAT provides on-demand SNAT ports for new outbound traffic flows. All available SNAT ports in inventory are used by any virtual machine on subnets configured with NAT. 

<p align="center">
  <img src="media/nat-overview/lb-vnnat-chart.svg" width="550" title="Virtual Network NAT on-demand outbound SNAT">
</p>

*Figure: Virtual Network NAT on-demand outbound SNAT*

Any IP configuration of a virtual machine can create outbound flows on-demand as needed.  Pre-allocation, per instance planning including per instance worst case overprovisioning, isn't required.  

<p align="center">
  <img src="media/nat-overview/exhaustion-threshold.svg" width="550" title="Differences in exhaustion scenarios">
</p>

*Figure: Differences in exhaustion scenarios*

Once a SNAT port releases, it's available for use by any virtual machine on subnets configured with NAT.  On-demand allocation allows dynamic and divergent workloads on subnet(s) to use SNAT ports as they need.  As long as there's SNAT port inventory available, SNAT flows will succeed. SNAT port hot spots benefit from the larger inventory instead. SNAT ports aren't left unused for virtual machines not actively needing them.

### Scaling

NAT needs sufficient SNAT port inventory for the complete outbound scenario. Scaling NAT is primarily a function of managing the shared, available SNAT port inventory. Sufficient inventory needs to exist to address the peak outbound flow for all subnets attached to a NAT gateway resource.

SNAT maps multiple private addresses to one public address and uses multiple public IPs to scale.

A NAT gateway resource will use 64,000 ports (SNAT ports) of a public IP address.  These SNAT ports become the available inventory for the private to public flow mapping. And adding more public IP addresses increases the available inventory SNAT ports. NAT gateway resources can scale up to 16 IP addresses and 1M SNAT ports.  TCP and UDP are separate SNAT port inventories and unrelated.

NAT gateway resources opportunistically reuse source ports. For scaling purposes, you should assume each flow requires a new SNAT port and scale the total number of available IP addresses for outbound traffic.

### Protocols

NAT gateway resources interact with IP and IP transport headers of UDP and TCP flows and are agnostic to application layer payloads.  Other IP protocols aren't supported.

### Timers

Idle timeout can be adjusted from 4 minutes (default) to 120 minutes (2 hours) for all flows.  Additionally, you can reset the idle timer with traffic on the flow.  A recommended pattern for refreshing long idle connections and endpoint liveness detection is TCP keepalives.  TCP keepalives appear as duplicate ACKs to the endpoints, are low overhead, and invisible to the application layer.

The following timers are used for SNAT port release:

| Timer | Value |
|---|---|
| TCP FIN | 60 seconds |
| TCP RST | 10 seconds |
| TCP half open | 30 seconds |

A SNAT port is available for reuse to the same destination IP address and destination port after 5 seconds.

>[!NOTE] 
>These timer settings are subject to change. The values are provided to help troubleshooting and you shouldn't take a dependency on specific timers at this time.

## Limitations

- NAT is compatible with standard SKU public IP, public IP prefix, and load balancer resources.   Basic resources (for example basic load balancer) and any products derived from them aren't compatible with NAT.  Basic resources must be placed on a subnet not configured with NAT.
- IPv4 address family is supported.  NAT doesn't interact with IPv6 address family.
- NSG on subnet or NIC isn't honored for outbound flows to public endpoints using NAT.
- NSG flow logging isn't supported when using NAT.
- When a virtual network has multiple subnets, each subnet can have a different NAT configured.
- NAT can't span multiple virtual networks.

## Preview participation

Follow [instructions to enable your subscription](nat-overview.md#public-preview-participation).

## Next steps

- Learn more about [virtual network NAT](nat-overview.md).
- Quickstart for deploying [NAT gateway resource using Azure CLI](quickstart-create-nat-gateway-cli.md).
- Quickstart for deploying [NAT gateway resource using Azure PowerShell](quickstart-create-nat-gateway-powershell.md).
- Quickstart for deploying [NAT gateway resource using Azure portal](quickstart-create-nat-gateway-portal.md).
- Learn more about [availability zones](../availability-zones/az-overview.md).
- Learn more about [standard load balancer](../load-balancer/load-balancer-standard-overview.md).
- Learn more about [availability zones and standard load balancer](../load-balancer/load-balancer-standard-availability-zones.md).
- Learn more about [NAT gateway resource REST API](https://docs.microsoft.com/rest/api/virtualnetwork/natgateways).
- Learn more about [NAT gateway resource Azure CLI](https://docs.microsoft.com/cli/azure/network/nat/gateway?view=azure-cli-latest).
- Learn more about [NAT gateway resource PowerShell](https://docs.microsoft.com/powershell/module/az.network/new-aznatgateway).
