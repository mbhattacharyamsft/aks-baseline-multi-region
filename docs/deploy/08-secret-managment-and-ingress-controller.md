# Configure AKS ingress controller with Azure Key Vault integration

Previously you have configured [the workload's prerequisites](./07-workload-prerequisites.md). The following steps configure Traefik, the AKS ingress solution used in this reference implementation, so that it can securely expose the web app to your Azure Application Gateway.

## Steps

1. Retrieve the details of the AKS ingress controller's managed identity.

   ```bash
   INGRESS_CONTROLLER_WORKLOAD_IDENTITY_CLIENT_ID_BU0001A0042_03=$(az deployment group show -g rg-bu0001a0042-03 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityClientId.value -o tsv)
   echo $INGRESS_CONTROLLER_WORKLOAD_IDENTITY_CLIENT_ID_BU0001A0042_03
   ```

1. Create the ingress controller's Secret Provider Class resource.

   The ingress controller exposes the wildcard TLS certificate you created in a prior step. It uses the Azure Key Vault CSI Provider to mount the certificate, which is managed and stored in Azure Key Vault. By mounting the certificate, Traefik can use it.
   
   The `SecretProviderClass` resource is created with reference to your federated identity and the Azure Key Vault parameters for the [Azure Key Vault Provider for Secrets Store CSI Driver](https://github.com/Azure/secrets-store-csi-driver-provider-azure).

   ```bash
   KEYVAULT_NAME_BU0001A0042_03=$(az deployment group show -g rg-bu0001a0042-03 -n cluster-stamp  --query properties.outputs.keyVaultName.value -o tsv)
   echo KEYVAULT_NAME_BU0001A0042_03: $KEYVAULT_NAME_BU0001A0042_03

   cat <<EOF | kubectl apply --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB -f -
   apiVersion: secrets-store.csi.x-k8s.io/v1
   kind: SecretProviderClass
   metadata:
     name: aks-ingress-contoso-com-tls-secret-csi-akv
     namespace: a0042
   spec:
     provider: azure
     parameters:
       clientID: $INGRESS_CONTROLLER_WORKLOAD_IDENTITY_CLIENT_ID_BU0001A0042_03
       usePodIdentity: "false"
       useVMManagedIdentity: "false"
       keyvaultName: $KEYVAULT_NAME_BU0001A0042_03
       objects:  |
         array:
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.crt
             objectType: cert
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.key
             objectType: secret
       tenantId: $TENANTID_AZURERBAC_AKS_MRB
   EOF
   ```

1. Install the Traefik ingress controller.

   Traefik will use the TLS certificate that was mounted by the CSI driver. Before going to production, we ensure the image reference comes from your Azure Container Registry by running the sed command that updates the `image:` value to reference your container registry instead of the default public container registry.

   ```bash
   sed -i -e "s/docker.io/${ACR_NAME_AKS_MRB}.azurecr.io/" workload/traefik-region1.yaml
   kubectl apply -f ./workload/traefik-region1.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
   ```

1. Wait for Traefik to be ready.

   During Traefik's pod creation process, the CSI driver communicates with Azure Key Vault to get the certificates needed for pod volume mount. This can take some time, but will eventually succeed if properly configured.

   ```bash
   kubectl wait -n a0042 --for=condition=ready pod --selector=app.kubernetes.io/name=traefik-ingress-ilb --timeout=90s --context $AKS_CLUSTER_NAME_BU0001A0042_03_AKS_MRB
   ```

1. Repeat the steps to install the ingress controller on your second AKS cluster, replacing the parameters that are different between the two clusters.

   ```bash
   # Get the AKS ingress controller managed identity details.
   INGRESS_CONTROLLER_WORKLOAD_IDENTITY_CLIENT_ID_BU0001A0042_04=$(az deployment group show -g rg-bu0001a0042-04 -n cluster-stamp --query properties.outputs.aksIngressControllerPodManagedIdentityClientId.value -o tsv)
   echo $INGRESS_CONTROLLER_WORKLOAD_IDENTITY_CLIENT_ID_BU0001A0042_04

   # Create the Traefik Secret Provider Class resource.
   KEYVAULT_NAME_BU0001A0042_04=$(az deployment group show -g rg-bu0001a0042-04 -n cluster-stamp  --query properties.outputs.keyVaultName.value -o tsv)
   echo KEYVAULT_NAME_BU0001A0042_04: $KEYVAULT_NAME_BU0001A0042_04

   cat <<EOF | kubectl apply --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB -f -
   apiVersion: secrets-store.csi.x-k8s.io/v1
   kind: SecretProviderClass
   metadata:
     name: aks-ingress-contoso-com-tls-secret-csi-akv
     namespace: a0042
   spec:
     provider: azure
     parameters:
       clientID: $INGRESS_CONTROLLER_WORKLOAD_IDENTITY_CLIENT_ID_BU0001A0042_04
       usePodIdentity: "false"
       useVMManagedIdentity: "false"
       keyvaultName: $KEYVAULT_NAME_BU0001A0042_04
       objects:  |
         array:
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.crt
             objectType: cert
           - |
             objectName: traefik-ingress-internal-aks-ingress-contoso-com-tls
             objectAlias: tls.key
             objectType: secret
       tenantId: $TENANTID_AZURERBAC_AKS_MRB
   EOF

   # Install the Traefik Ingress Controller in the second region.
   sed -i -e "s/docker.io/${ACR_NAME_AKS_MRB}.azurecr.io/" workload/traefik-region2.yaml
   kubectl apply -f ./workload/traefik-region2.yaml --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB

   # Wait for Traefik to be ready.
   kubectl wait --namespace a0042 --for=condition=ready pod --selector=app.kubernetes.io/name=traefik-ingress-ilb --timeout=90s --context $AKS_CLUSTER_NAME_BU0001A0042_04_AKS_MRB
   ```

### Next step

:arrow_forward: [Deploy the workload](./09-workload.md)
