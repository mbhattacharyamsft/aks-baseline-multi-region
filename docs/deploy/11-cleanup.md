# Clean up

After you are done exploring your deployed [AKS baseline multi cluster](/README.md), you'll want to delete the created Azure resources to avoid incurring unnecessary costs. Follow these steps to delete all resources that you created as part of this reference implementation.

## Steps

1. Delete the resource groups, which deletes all of the Azure resources they contain.

   :warning: Ensure you are using the correct subscription, and verify that you didn't create any resources in the groups that you want to retain.

   ```bash
   az group delete -n rg-bu0001a0042-03
   az group delete -n rg-bu0001a0042-04
   az group delete -n rg-enterprise-networking-spokes
   az group delete -n rg-enterprise-networking-hubs
   az group delete -n $SHARED_RESOURCE_GROUP_NAME_AKS_MRB
   ```

1. Purge your key vault.

   Because this reference implementation enables soft delete on Key Vault, execute a purge so your next deployment of this implementation doesn't run into a naming conflict.

   ```bash
   az keyvault purge -n $KEYVAULT_NAME_BU0001A0042_03
   az keyvault purge -n $KEYVAULT_NAME_BU0001A0042_04
   ```

1. Manually delete the flow log resources.

   The `NetworkWatcherRG` resource group is where flow log definitions were created for this reference implementation. All of the flow logs that were created were prefixed with `fl` and were followed by a GUID, targeting a virtual network either in the hub or spokes resource group.

1. [Remove the Azure Policy assignments](https://portal.azure.com/#blade/Microsoft_Azure_Policy/PolicyMenuBlade/Compliance) that are scoped to the cluster's resource group. To identify those created by this implementation, look for policy assignments that are prefixed with `[your-cluster-name]`.

   Alternatively execute the following commmands:

   ```bash
   for p in $(az policy assignment list --disable-scope-strict-match --query "[?resourceGroup=='rg-bu0001a0042-03'].name" -o tsv); do az policy assignment delete --name ${p} --resource-group rg-bu0001a0042-03; done
   for p in $(az policy assignment list --disable-scope-strict-match --query "[?resourceGroup=='rg-bu0001a0042-04'].name" -o tsv); do az policy assignment delete --name ${p} --resource-group rg-bu0001a0042-04; done
   ```

1. If any temporary changes were made to Microsoft Entra ID or Azure RBAC permissions, consider removing those as well.

### Next step

:arrow_forward: [Review additional information in the main README](/README.md#broom-clean-up-resources)
