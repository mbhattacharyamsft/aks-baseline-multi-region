# Generate your client-facing and AKS ingress controller TLS certificates

Now that the [hub-spoke network is provisioned](./04-networking.md), you can create TLS certificates that your solution needs. You'll create multiple TLS certificates: one for each region that Azure Front Door routes to an Application Gateway instance, for clients connecting to your web app; and another for the AKS ingress controller.

## Expected results

The following steps will create the certificates needed for Azure Application Gateway and AKS ingress controller.

| Certificate type | Purpose |
|:- |:- |
| Two Azure Application Gateway certificates | TLS certificates issued by Let's Encrypt for the public IP FQDNs and served by the Azure Application Gateway instances in each region. |
| An AKS ingress controller certificate | A self-signed wildcard certificate for TLS on the cluster ingress controller. |

## Steps

1. Generate client-facing TLS certificates for each region.

   > :book: The Contoso Bicycle organization has an important policy that every internet-facing endpoint exposed over the HTTPS protocol must use a trusted CA certificate, and it is not allowed to share a common wildcard certificate between them. Therefore, the organization needs to procure individual trusted CA certificates for all their Public IPs' FQDNs in the different regions. The Azure Application Gateway instances are going to be serving these certificates in front of every region they get deployed to.

   ```bash
   # call the Let's Encrypt certificate generation script for both PIPs' FQDNs
   # [Generating the certificates takes about twenty minutes to run.]
   chmod +x ./certs/letsencrypt-pip-cert-generation.sh ./certs/authenticator.sh
   ./certs/letsencrypt-pip-cert-generation.sh $(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0042-03 --query properties.outputs.appGatewayPublicIp.value -o tsv)
   ./certs/letsencrypt-pip-cert-generation.sh $(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0042-04 --query properties.outputs.appGatewayPublicIp.value -o tsv)
   ```

   :bulb: Extended validation (EV) certificates are mostly recommended for user-facing endpoints. This multiregion reference implementation doesn't generate certificates for user-facing endpoints. The certificates generated here are for communication between Azure Front Door and Azure Application Gateway. For more information on how these certificates are generated, please refer to [Certificate Generation for an Azure Public IP with your DNS Prefix](https://github.com/mspnp/letsencrypt-pip-cert-generation).

1. Base 64-encode the client-facing certificates.

   :bulb: Whether you use a certificate from your organization or you generated one in the preceding step, you'll need the certificate's PFX file to be base 64-encoded so that it can be stored in Azure Key Vault later in this process.

   ```bash
   # get the public IP address subdomains
   APPGW_SUBDOMAIN_BU0001A0042_03=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0042-03 --query properties.outputs.subdomainName.value -o tsv)
   APPGW_SUBDOMAIN_BU0001A0042_04=$(az deployment group show -g rg-enterprise-networking-spokes -n spoke-BU0001A0042-04 --query properties.outputs.subdomainName.value -o tsv)
   echo APPGW_SUBDOMAIN_BU0001A0042_03: $APPGW_SUBDOMAIN_BU0001A0042_03
   echo APPGW_SUBDOMAIN_BU0001A0042_04: $APPGW_SUBDOMAIN_BU0001A0042_04

   export APP_GATEWAY_LISTENER_REGION1_CERTIFICATE_BASE64_AKS_MRB=$(cat ${APPGW_SUBDOMAIN_BU0001A0042_03}.pfx | base64 | tr -d '\n')
   export APP_GATEWAY_LISTENER_REGION2_CERTIFICATE_BASE64_AKS_MRB=$(cat ${APPGW_SUBDOMAIN_BU0001A0042_04}.pfx | base64 | tr -d '\n')
   echo APP_GATEWAY_LISTENER_REGION1_CERTIFICATE_BASE64_AKS_MRB: $APP_GATEWAY_LISTENER_REGION1_CERTIFICATE_BASE64_AKS_MRB
   echo APP_GATEWAY_LISTENER_REGION2_CERTIFICATE_BASE64_AKS_MRB: $APP_GATEWAY_LISTENER_REGION2_CERTIFICATE_BASE64_AKS_MRB
   ```

1. Generate a wildcard certificate for the AKS ingress controller.

   > :book: Contoso Bicycle will also procure a standard TLS certificate to be used with the AKS ingress controller. This certificate is not EV, because it won't be user-facing. Finally the app team decides to use a wildcard certificate with the wildcard `*.aks-ingress.contoso.com` for the ingress controller. Because this is not an internet-facing endpoint; using a wildcard certificate is a valid option that meets Contoso Bicycle's policies.

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out traefik-ingress-internal-aks-ingress-contoso-com-tls.crt -keyout traefik-ingress-internal-aks-ingress-contoso-com-tls.key -subj "/CN=*.aks-ingress.contoso.com/O=Contoso Aks Ingress"

   # Combine into a PEM structure, which is required by Azure Application Gateway for backend pools
   cat traefik-ingress-internal-aks-ingress-contoso-com-tls.crt traefik-ingress-internal-aks-ingress-contoso-com-tls.key > traefik-ingress-internal-aks-ingress-contoso-com-tls.pem
   ```

1. Base 64-encode the AKS ingress controller certificate.

   :bulb: Whether you use a certificate from your organization or you generated one from above, you'll need the public certificate's `.crt` or `.cer` file to be base 64-encoded so that it can be stored in Azure Key Vault later in this process.

   ```bash
   export AKS_INGRESS_CONTROLLER_CERTIFICATE_BASE64_AKS_MRB=$(cat traefik-ingress-internal-aks-ingress-contoso-com-tls.crt | base64 | tr -d '\n')
   echo AKS_INGRESS_CONTROLLER_CERTIFICATE_BASE64_AKS_MRB: $AKS_INGRESS_CONTROLLER_CERTIFICATE_BASE64_AKS_MRB
   ```

### Save your work in-progress

```bash
# run the saveenv.sh script at any time to save environment variables created above to aks_baseline.env
./saveenv.sh

# if your terminal session gets reset, you can source the file to reload the environment variables
# source aks_baseline.env
```

### Next step

:arrow_forward: [Deploy the AKS clusters](./06-aks-cluster.md)
