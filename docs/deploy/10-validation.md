# Validate the end-to-end multi-cluster architecture

Now that you have a workload deployed in multiple regions, you can start validating and exploring this reference implementation of the [AKS baseline multi cluster](/README.md). In addition to validating the workload itself, you can also perform validation on some of the observability components of the workload.

## Validate the region failover process

This section validates the workload is exposed correctly and is responding to incoming HTTP requests. Additionally, the workload is now deployed into multiple regions, and you want to test the failover.

### Steps

1. Retrieve your Azure Application Gateway instances names.

   ```bash
   APPGW_BU0001A0042_03=$(az deployment group show -g rg-bu0001a0042-03 -n cluster-stamp --query properties.outputs.agwName.value -o tsv)
   APPGW_BU0001A0042_04=$(az deployment group show -g rg-bu0001a0042-04 -n cluster-stamp --query properties.outputs.agwName.value -o tsv)
   echo APPGW_BU0001A0042_03: $APPGW_BU0001A0042_03
   echo APPGW_BU0001A0042_04: $APPGW_BU0001A0042_04
   ```

1. Retrieve your Azure Front Door endpoint's public DNS name.

   > :book: The app team conducts a final system test to be sure that traffic is flowing end-to-end as expected. They place some requests against the Azure Front Door endpoint.

   ```bash
   # query the Azure Front Door endpoint FQDN
   FRONTDOOR_FQDN=$(az deployment group show -g $SHARED_RESOURCE_GROUP_NAME_AKS_MRB -n shared-svcs-stamp --query properties.outputs.fqdn.value -o tsv)
   echo FRONTDOOR_FQDN: $FRONTDOOR_FQDN
   ```

1. Manually load the endpoint in a web browser. Ensure that you specify `https://` at the start of the FQDN.

   > :book: The app team configured the Azure Front Door origin group to balance the traffic equally between the two regions. Azure Front Door routes traffic based on the roundtrip latency between the Azure Front Door PoP closest to the user and the origins.

1. Add some load to the web application to test that it consistently responds with a 200 response code, even while the system is experiencing (simulated) regional outages.

   ```bash
   for i in {1..200}; do curl -I https://$$FRONTDOOR_FQDN && sleep 10; done
   ```

   > :eyes: The above script sends one HTTP request every ten seconds to your infrastructure. In total, 200 HTTP requests will be sent over about 33 minutes.

1. Open another terminal instance. You'll use this terminal to stop and restart the application gateway instances as a way to simulate total regional outages and regional recovery. Observe how your application is still responsive throughout the tests.

   > :book: The app team wants to run some simulations for outages in both the East US 2 and Central US regions. They want to ensure both regions can fail over to each other under such demanding circumstances.

   > :eyes: After each of the commands in the following script are executed, you should immediately return to your previous terminal and observe that the web application is responding with `HTTP 200` even during the outages.

   ```bash
   APPGW_BU0001A0042_03=$(az deployment group show -g rg-bu0001a0042-03 -n cluster-stamp --query properties.outputs.agwName.value -o tsv)
   APPGW_BU0001A0042_04=$(az deployment group show -g rg-bu0001a0042-04 -n cluster-stamp --query properties.outputs.agwName.value -o tsv)
   echo APPGW_BU0001A0042_03: $APPGW_BU0001A0042_03
   echo APPGW_BU0001A0042_04: $APPGW_BU0001A0042_04

   # [This whole execution takes about 40 minutes.]
   az network application-gateway stop -g rg-bu0001a0042-03 -n $APPGW_BU0001A0042_03 && \ # first incident
     az network application-gateway start -g rg-bu0001a0042-03 -n $APPGW_BU0001A0042_03 && \
     az network application-gateway stop -g rg-bu0001a0042-04 -n $APPGW_BU0001A0042_04 && \ # second incident
     az network application-gateway start -g rg-bu0001a0042-04 -n $APPGW_BU0001A0042_04
   ```

   :bulb: Remember that after a total failover, your infrastructure in the remaining region must be capable of handling 100% of the traffic. Therefore, the recommendation is to plan for this sort of event, and prepare all of your other backend systems to cope with traffic influx as well. You don't necessarily need to purchase resources to keep them idle waiting for these very rare outages. But you want to cover every piece of infrastructure to ensure they are elastic (scalable) enough to handle any sudden increase in load. For example, [Azure Application Gateway v2 autoscaling](https://learn.microsoft.com/azure/application-gateway/application-gateway-autoscaling-zone-redundant) is initiated under these circumstances. Similarly, the Cluster Autoscaler configured as part of the [AKS baseline](https://github.com/mspnp/aks-baseline) also automatically scales when load increases. Consider setting up [Horizontal Pod Autoscaling](https://learn.microsoft.com/azure/aks/concepts-scale#horizontal-pod-autoscaler) and [Resource Quotas](https://learn.microsoft.com/azure/aks/operator-best-practices-scheduler#enforce-resource-quotas). The latter must be configured to allow up to 2X the normal capacity if you plan for one region to completely take the traffic of another region during an outage. To avoid cascade failures, consider any out-of-cluster resources that are affected by your workload, such as databases, external APIs, and so on. They also must be ready to absorb as much load as you consider appropriate in your regional outage contingency plan.

## Azure Monitor dashboard

Azure Monitor for Containers, as well as other metrics published by other Azure resources, enables you to deeply observe your infrastructure. It's a good idea to create a dashboard on top of the underlying monitoring data. This dashboard should enable an organization's SRE team to quickly verify that everything is healthy. If something is in a degraded state, a dashboard enables quick investigation by getting more details from the resources. The idea to create a clean, organized and accessible view of your infrastructure.

We don't deploy a dashboard as part of this reference implementation, but we encourage you to create your own as you see fit, based on the metrics that matter to your system.

In the following sections, we describe an example afternoon where regions fail in different ways.

### Incident 1: 4:44 PM UTC time, East US 2 is in trouble

Traffic is being handled by East US 2 because this is closest region to the client sending HTTP requests. As detailed above, Azure Front Door routes all the traffic to the fastest origin measured by their latency to the user's PoP. It is around 4:44 PM when the region outage happens.

Take a look at how the East US 2 application gateway's healthiness drops to 27%, and its compute units drop close to 0. This indicates a worst-case scenario - a complete shutdown.

> :warning: Depending on your actual location traffic might flow different for you. But having two simulated incidents ensures that you experience at least one failover.

The Central US instance rescues the situation. We expect to lose just a few packets before traffic is rerouted. The architecture was designed to be highly available, so this incident was a transparent experience for your clients, and the traffic remained flowing throughout the incident without inconvenience.

![Azure Monitor Dashboard that helps to observe the East US 2 region outage simulation and how the traffic flowed from East US 2 to Central US](images/azure-monitor-dashboard-1st-failover.png)

> Note: Based on the inbound multi-cluster traffic flow count and the Application Gateway Health Dashboard Metrics: :large_blue_circle: East US 2 :red_circle: Central US

### Incident 2: 4:56 PM UTC time, East US 2 resumes its operations but Central US is about to go down

Now East US 2 region is back after an outage of approximately 12 minutes. Every 30 seconds, Azure Front Door samples the roundtrip latencies against its backend pools using the configured health probe, and once again determines East US 2 is the best candidate as this instance normalized its operations. Traffic starts flowing at 4:56 PM the other way around from Central US to East US 2, and now everything is back to normal.

At the same time, you observe a new outage with the same symptoms as before, but it is now in Central US. Once again the compute units are close to 0 and healthiness is about to drop to 0% a few seconds later. It does not represent a threat since just a moment ago traffic had already flowed to East US 2.

Around 5:15 PM all regions are operative, and the clients never suffered the consequences of these multiple total region outages.

![Azure Monitor Dashboard that helps to observe the Central US stops serving in favor of East US 2 region as this is back from first incident. It displays the traffic flowing now the other way around from Central US to East US 2](images/azure-monitor-dashboard-back-to-normal.png)

> Note: Based on the inbound multi-cluster traffic flow count and the Application Gateway Health Dashboard Metrics: :large_blue_circle: East US 2 :red_circle: Central US

## Validate the centralized Azure Log Analytics workspace logs

Each cluster's logs are stored in the same `ContainerLogsV2` table within the shared Log Analytics workspace. In the case of your workload, you could also include additional logging and telemetry frameworks, such as Application Insights. Here are the steps to view the built-in application logs:

1. In the Azure portal, navigate to your AKS cluster resource group (`rg-bu0001a0042-shared-\<region>`).
1. Select your Log Analytics Workspace resource.
1. Navigate under General and click Logs. Then execute the following query

   ```kusto
   ContainerLogV2
   | project TimeGenerated, LogMessage, Computer, ContainerName, ContainerId
   | order by TimeGenerated desc
   ```

## Traffic Analytics

> :book: The networking team is now dealing with multiple clusters in different regions. Understanding how the traffic flows at layers 4 and 7 through their deployed networking topology is now more critical than ever. That's why the team is evaluating different tooling that could provide monitoring over their networks. One of the Azure Monitor products is Network Watcher, which offers two really interesting features: NSG flow logs, and with it, [Traffic Analytics](https://learn.microsoft.com/azure/network-watcher/traffic-analytics).
>
> Traffic Analytics can analyze important information of traffic, including where it originates from, how it flows thought the different regions, and how much of the traffic is benign or malicious. It includes many more details at the security and performance level. [With no upfront cost and no termination fees](https://azure.microsoft.com/pricing/details/network-watcher/) the business unit (BU0001) would be charged for collection and processing logs per GB at 10-min or 60-min intervals.

Network security group flow logs were configured as part of the default installation of this reference implementation. Along with the raw logs being stored in the storage account in the hub resource group, you can access Azure Traffic Analytics for a dashboard of network flows:

1. Open the [Azure Traffic Analytics hub](https://portal.azure.com/#blade/Microsoft_Azure_Network/NetworkWatcherMenuBlade/trafficAnalytics) in the Azure portal.
1. Adjust filters as needed at the top.
1. You can then see application ports, NSG hits, application gateway, and other network flows.

![Traffic Analytics geo-map view of the multi-cluster reference implementation while under load. Traffic is coming from a single Azure Front Door PoP and is distributed to both regions after the first failover is complete](./images/traffic-analytics-geo-map.gif)

## Next step

:arrow_forward: [Clean Up Azure Resources](./11-cleanup.md)
