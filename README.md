# Understanding ExpressRoute private peering to address ExpressRoute resiliency

This article is co-authored by Microsoft colleague [David Santiago](https://github.com/davidsntg). The content is now also available on the  [Microsoft Tech Community blog](https://techcommunity.microsoft.com/t5/azure-networking-blog/understanding-expressroute-private-peering-to-address/ba-p/4081850), go and check it out!


- [Scope](#scope)
- [1. ExpressRoute components](#1-expressroute-components)
- [2. ExpressRoute models](#2-expressroute-models)
  * [2.1. ExpressRoute Service Provider models](#21-expressroute-service-provider-models)
    + [Cloud exchange colocation](#cloud-exchange-colocation)
    + [Ethernet Point to Point](#ethernet-point-to-point)
    + [Any-to-any connectivity](#any-to-any-connectivity)
  * [2.2. ExpressRoute Direct model](#22-expressroute-direct-model)
- [3. What could go wrong?](#3-what-could-go-wrong) 
  * [3.1. ExpressRoute peering location failure](#31-expressroute-peering-location-failure)
    + [Solution #1: geo-redundant ExpressRoute circuits](#solution-1-geo-redundant-expressroute-circuits) 
    + [Solution #2: S2S VPN backup](#solution-2-s2s-vpn-backup) 
  * [3.2. On-Prem misconfigurations/failures and MSEE maintenances](#32-on-prem-misconfigurationsfailures-and-msee-maintenances) 
  * [3.3. Availability Zone Failure](#33-availability-zone-failure)
  * [3.4. Max advertised prefix limit exceeded](#34-max-advertised-prefix-limit-exceeded)

# Scope

> No breaking news here, just an illustrated recap of the recommendations and attention points highlighted here and there in the [Microsoft ExpressRoute documentation](https://learn.microsoft.com/en-us/azure/expressroute/). 

This article focuses on [ExpressRoute](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-introduction) Private Peering only, to connect an On-Prem network and VNets in an Azure hub-and-spoke or Virtual WAN environment.

ExpressRoute connectivity is provided in [ExpressRoute peering locations](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-locations). ExpressRoute peering locations are entry points into the Microsoft backbone, [Azure regions](https://azure.microsoft.com/en-us/explore/global-infrastructure/geographies/#overview) are where the Azure resources are hosted: distinct concepts at different locations. 

 
# 1. ExpressRoute components

3 main components: the Circuit, the Gateway and the Connection.

![](<images/er-components.png>)

| **Components** | **Connectivity** | **Location** |
|---|---|---|
|ExpressRoute Circuit|**Dual** physical fiber connectivity between the MSEEs and the provider equipments|Expressroute peering location|
|ExpressRoute Gateway|Min 2 instances connected to both MSEEs| Azure region|
|ExpressRoute Connection|Virtual connection between the MSEE and the ExpressRoute Gateway|ExpressRoute location to Azure region|

The provider must ensure redundant connectivity to either the customer edge routers or their MPLS edge routers.


# 2. ExpressRoute models

There are 4 [ExpressRoute connectivity models](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-connectivity-models) between On-Prem and Azure, divided in 2 approaches: 3 ExpressRoute ***Service Provider*** models and 1 ExpressRoute ***Direct*** model.

## 2.1. ExpressRoute Service Provider models

In these models, ExpressRoute connectivity is provided to customers through [service providers](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-locations-providers#partners) and can be delivered in 3 different ways.

### Cloud exchange colocation

The customer edge routers are located **in a cloud exchange facility** near to or at the peering location and are cross-connected with an ExpressRoute connectivity provider using either L2 or L3.

![](images/cloud-exch-colo.png)

### Ethernet Point to Point

The customer edge routers **at a branch** are connected via point-to-point Ethernet links to an ExpressRoute connectivity provider, utilizing either L2 or L3.

![](images/eth-p2p.png)

### Any-to-any connectivity

In this scenario, ExpressRoute is associated with a **customer VRF** within the WAN provider network, making Azure appear as just any other branch connected to the customer's MPLS backbone. The routers of the ExpressRoute connectivity provider are cross-connected with colocated WAN service provider routers.

![](images/any2any.png)

## 2.2. ExpressRoute Direct model

[ExpressRoute Direct](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-erdirect-about) is a dedicated physical connection to the Microsoft backbone, established between a pair of MSEEs and customer routers, without any intermediate connectivity provider. The customer is allocated an entire MSEE port (10 Gbps or 100 Gbps) allowing the creation of multiple circuits on it.

![](images/erd.png)

Adam Stuart's [video](https://youtu.be/Yk5bFWhdVJg?si=tjix2A-ity2sLmjp) is another valuable resource for learning about ExpressRoute Direct.

# 3. What could go wrong?

## 3.1. ExpressRoute peering location failure

![](images/er-peering-location-failure.png)

### Solution #1: geo-redundant ExpressRoute circuits

To address ExpressRoute peering location failures, the recommended solution is to build a resilient design as outlined in [this article](https://learn.microsoft.com/en-us/azure/expressroute/designing-for-disaster-recovery-with-expressroute-privatepeering) and illustrated below:

![](images/er-circuit-resiliency.png)

Resiliency is achieved by deploying 2 ExpressRoute Circuits in 2 distinct ExpressRoute peering locations, thereby creating geo-redundant ExpressRoute circuits that are both connected to the same Azure ExpressRoute Gateway.

Based on this principle, environments with multiple Azure regions provide the opportunity to leverage existing ExpressRoute Circuits to achieve geo-redundancy through an ***ExpressRoute Bow-Tie*** design:

![](images/er-bowtie.png)

> Because these designs introduce 2 parallel paths to Azure, traffic engineering mechanisms must be carefully implemented to prevent unexpected asymmetric routing. 

To create an optimal geo-redundant ExpressRoute Circuit design, it is important to understand that the ExpressRoute Circuit SKU determines where an ExpressRoute circuit can connect to. Each SKU has a specific scope of supported regions (same metro/same geo/cross-geo):

![](images/er-circuit-skus.png)

See [this great repo](https://github.com/Danieleg82/Exr-Bowtie) from Daniele Gaiulli to learn more about the the benefits of Expressroute bow-tie designs.

### Solution #2: S2S VPN backup

ExpressRoute and VPN can also be combined in an Active-Passive configuration to provide disaster recovery capabilities ([doc](https://learn.microsoft.com/en-us/azure/expressroute/use-s2s-vpn-as-backup-for-expressroute-privatepeering)).

![](images/s2svpn-backup.png)

* Both a VPN Gateway and an ExpressRoute Gateway can exist in the same `GatewaySubnet`. 
* Transit routing between the ExpressRoute Gateway and the VPN Gateway is not possible without the use of [ARS](https://learn.microsoft.com/en-us/azure/route-server/overview) or [Virtual WAN](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-about).
* ExpressRoute routes take precedence over other routes for the same prefixes.
* When different but overlapping prefixes are used, the route with the longest prefix is chosen.

## 3.2. On-Prem misconfigurations/failures and MSEE maintenances

![](images/er-msee-maintenance.png)

During an ExpressRoute Circuit maintenance, one link out of the 2 fibers connecting the MSEEs and provider equipments remains available. AS-prepending is used by Microsoft to force traffic over the remaining link. 

To prevent conflicts between Microsoft AS-prepending and On-Prem routing and to maintain On-Prem network resiliency, it is important to plan for this scenario and operate both links of an ER circuit (highlighted in yellow) as [Active/Active](https://learn.microsoft.com/en-us/azure/expressroute/designing-for-high-availability-with-expressroute#active-active-connections) when in nominal mode. 

![](images/active-active.png)

To prevent a single point of failure, it also recommended to terminate the primary and the secondary links of an ExpressRoute circuit on 2 separate customer routers as highlighted in orange on the above diagram.

More details in [this video](https://www.youtube.com/watch?v=CuXOszhSWjc).

## 3.3. Availability Zone Failure

![](images/er-az-failure.png)

The ExpressRoute Gateway instances in Azure connect VNets and MSEE routers by handling BGP route exchanges between them.

Therefore, the resiliency of the ExpressRoute Gateway is crucial to ensure end-to-end ExpressRoute resiliency.  It is best practice to use [AZ-redundant ExpressRoute Gateway SKUs](https://learn.microsoft.com/en-us/azure/expressroute/expressroute-about-virtual-network-gateways#zrgw) for this purpose. 

Comparison table:

| **Regular ERGW SKUs** | **Zone-redundant ERGW SKUs** |
|---|---|
|Basic (deprecated): 500 Mbps throughput between VNet and MSEE | |
|Standard: 1 Gbps throughput|**ErGW1AZ**: 1 Gbps throughput|
|High Performance: 2 Gbps throughput|**ErGW2AZ**: 2 Gbps throughput|
|Ultra Performance: 10 Gbps throughput|**ErGW3AZ**: 10 Gbps throughput|

AZ-redundant ExpressRoute Gateway instances are distributed across different Availability Zones:

![](images/az-redundant-ergw.png)

## 3.4. Max advertised prefix limit exceeded

As per [Route Advertisement limits documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#route-advertisement-limits), it is important to note that the max number of routes advertised from Azure to On-Prem **is not the same** as from On-Prem to Azure.

![](images/route-advertisement-limits.png)

[Custom alerts](https://learn.microsoft.com/en-us/azure/expressroute/how-to-custom-route-alert) can be configured to monitor advertised routes over ExpressRoute.

 It is essential to consider the ExpressRoute Circuit SKU and the specific route advertisement limit it offers while planning your network architecture.

# Additional resources

* Common ExpressRoute design errors [in video](https://www.youtube.com/watch?v=b1CEte07IqA).
