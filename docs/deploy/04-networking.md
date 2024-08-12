# Deploy the hub-spoke network topology

In the prior step, you configured your Microsoft Entra tenant to create the [shared resources](./03-cluster-prerequisites.md) you need for this reference implementation deployment. Now we deploy the network resources.

## Subscription and resource group topology

This reference implementation is split across several resource groups in a single subscription. This arrangement replicates the fact that many organizations split certain responsibilities into specialized subscriptions (such as regional hubs/VWAN in a *Connectivity* subscription and workloads in landing zone subscriptions). We expect you to explore this reference implementation within a single subscription, but when you implement this cluster at your organization, you will need to take what you've learned here and apply it to your expected subscription and resource group topology. For a production solution, use the subscription organization principles outlined in [the Cloud Adoption Framework](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-subscriptions). The single-subscription, multiple-resource group model we use in this example is for simplicity of demonstration purposes only.

## Expected results

### Resource groups

In the following steps, these two resource groups will be created and populated with networking resources:

| Name | Purpose |
|:- |:- |
| `rg-enterprise-networking-hubs` | Contains all of your organization's regional hub networks. Each regional hub includes an egress firewall and Log Analytics for network logging. |
| `rg-enterprise-networking-spokes` | Contains all of your organization's regional spoke networks and related networking resources. All spokes will peer with their regional hub and subnets will egress through the regional firewall in the hub. |

### Resources

- One Azure Firewall in each region's hub virtual network
- One spoke virtual network in each region
- Network peering from each spoke to its corresponding regional hub
- User-defined route to force tunnel traffic from cluster subnets to their hubs
- Network security groups for all subnets that support them

## Steps

1. Create resource groups for the hub networks and spoke networks.

   > :book: The networking team has all their regional networking hubs and spokes in the following centrally-managed Azure resource groups.

   ```bash
   az group create -n rg-enterprise-networking-hubs -l centralus
   az group create -n rg-enterprise-networking-spokes -l centralus
   ```

   :bulb: These resource groups would typically already exist.

1. Create two hub networks, and two spoke networks that will contain the AKS clusters and associated resources. Then, peer the spokes with the hubs.

   > :book: The networking team had created two generic hub networks that were waiting for customers to join. They receive a request from an app team in business unit (BU) 0001. The request is for the creation of network spokes to house their new AKS-based application (internally know as Application ID: A0042). The networking team talks with the app team to understand their requirements and aligns those needs with Microsoft's best practices for a secure AKS cluster deployment. As part of the non-functional requirements, the app team mentions they need to run two separate infrastructure instances of the same application from two different regions, so they can increase the availability. The networking team realizes they are going to need two different spokes to fulfil the app team's requirements. They capture those specific requirements and deploy the spokes (`BU0001A0042-03` and `BU0001A0042-04`), aligning to those specs, and connect each spoke to the corresponding regional hub network.
   >
   > The networking team has decided that `10.200.[0-9].0` will be where all regional hubs are homed on their organization's network space. The `eastus2` and `westus2` hubs (created below) will be `10.200.3.0/24` and `10.200.4.0/24` respectively.
   >
   > Note: The subnets for Azure Bastion and cross-premises connectivity are deployed in this reference architecture, but the resources aren't deployed. This reference implementation is expected to be isolated from existing infrastructure, so these IP addresses shouldn't conflict with any existing networking you have, even if those IP addresses overlap. If you need to connect the reference implementation to existing networks, you need to adjust the IP space as per your requirements as to not conflict in the reference Bicep files.

   The Azure Firewall base policies for the Contoso organization were created by the networking team as another shared resource. This way, they became available for each regional stamp that requires them. The stamp can inherit the policies and create child Azure Firewall policy rules on top of them. An important Azure Resource Manager requirement at the time writing this is that all derivative Azure Firewall Policies must reside in the same location as the parent.

   ```bash
   # [Creating the generic hubs takes about ten minutes to run (each).]
   BASE_FIREWALL_POLICIES_ID=$(az deployment group show -g $SHARED_RESOURCE_GROUP_NAME_AKS_MRB -n shared-svcs-stamp --query properties.outputs.baseFirewallPoliciesId.value -o tsv)
   echo BASE_FIREWALL_POLICIES_ID: $BASE_FIREWALL_POLICIES_ID

   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-region.v1.bicep -n hub-regionA -p baseFirewallPoliciesId=$BASE_FIREWALL_POLICIES_ID firewallPolicyLocation=eastus2 @networking/hub-region.parameters.eastus2.json
   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-region.v1.bicep -n hub-regionB -p baseFirewallPoliciesId=$BASE_FIREWALL_POLICIES_ID firewallPolicyLocation=eastus2 @networking/hub-region.parameters.centralus.json

   # [Creating the spokes takes about ten minutes to run.]
   RESOURCEID_VNET_HUB_REGIONA=$(az deployment group show -g rg-enterprise-networking-hubs -n hub-regionA --query properties.outputs.hubVnetId.value -o tsv)
   RESOURCEID_VNET_HUB_REGIONB=$(az deployment group show -g rg-enterprise-networking-hubs -n hub-regionB --query properties.outputs.hubVnetId.value -o tsv)
   echo RESOURCEID_VNET_HUB_REGIONA: $RESOURCEID_VNET_HUB_REGIONA
   echo RESOURCEID_VNET_HUB_REGIONB: $RESOURCEID_VNET_HUB_REGIONB

   az deployment group create -g rg-enterprise-networking-spokes -f networking/spoke-BU0001A0042.bicep -n spoke-BU0001A0042-03 -p hubVnetResourceId="${RESOURCEID_VNET_HUB_REGIONA}" @networking/spoke-BU0001A0042.parameters.eastus2.json
   az deployment group create -g rg-enterprise-networking-spokes -f networking/spoke-BU0001A0042.bicep -n spoke-BU0001A0042-04 -p hubVnetResourceId="${RESOURCEID_VNET_HUB_REGIONB}" @networking/spoke-BU0001A0042.parameters.centralus.json

   # [Enrolling the spokes into the hubs takes about ten minutes to run (each).]
   RESOURCEID_SUBNET_NODEPOOLS_BU0001A0042_03=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0042-03 --query properties.outputs.nodepoolSubnetResourceIds.value -o tsv)
   RESOURCEID_SUBNET_NODEPOOLS_BU0001A0042_04=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0042-04 --query properties.outputs.nodepoolSubnetResourceIds.value -o tsv)
   echo RESOURCEID_SUBNET_NODEPOOLS_BU0001A0042_03: $RESOURCEID_SUBNET_NODEPOOLS_BU0001A0042_03
   echo RESOURCEID_SUBNET_NODEPOOLS_BU0001A0042_04: $RESOURCEID_SUBNET_NODEPOOLS_BU0001A0042_04

   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-region.v1.1.bicep -n hub-regionA -p nodepoolSubnetResourceIds="['${RESOURCEID_SUBNET_NODEPOOLS_BU0001A0042_03}']" baseFirewallPoliciesId=$BASE_FIREWALL_POLICIES_ID firewallPolicyLocation=eastus2 @networking/hub-region.parameters.eastus2.json
   az deployment group create -g rg-enterprise-networking-hubs -f networking/hub-region.v1.1.bicep -n hub-regionB -p nodepoolSubnetResourceIds="['${RESOURCEID_SUBNET_NODEPOOLS_BU0001A0042_04}']" baseFirewallPoliciesId=$BASE_FIREWALL_POLICIES_ID firewallPolicyLocation=eastus2 @networking/hub-region.parameters.centralus.json
   ```

## Prepare for a failover

The [AKS baseline](https://github.com/mspnp/aks-baseline) describes the current [network topology segmentation](https://github.com/mspnp/aks-baseline/blob/main/networking/topology.md).

When you're designing an active/active architecture, remember that each network needs to be right-sized to absorb a sudden increase in traffic that might request twice the number of IPs when scheduling more *pods* to handle failover of a region.

### Next step

:arrow_forward: [Generate your TLS certificates](./05-ca-certificates.md)
