# Introduction
Demo setup for the **"Securing your app's communications with Kubernetes, Azure Key Vault, and TLS certificates"** talk.

More information about the workings of the Kubernetes Secrets Store CSI Driver on my blog:

https://damn.engineer/2022/01/31/azure-keyvault-to-kubernetes

and

https://damn.engineer/2022/02/07/tls-cert-azure-keyvault-kubernetes

# Pre-requisites
* [Azure subscription](https://azure.microsoft.com/en-us/free/)
* [Docker](https://docs.docker.com/get-docker/)
* [Minikube](https://minikube.sigs.k8s.io/docs/start/)
* [Helm](https://helm.sh/docs/intro/install/)

# Instructions
Make sure to run all commands from the root of the repository.

## Azure setup
Key Vaults need to have a globally unique name. This should generate an unused name:
```
export kv="ksscd-demo-$RANDOM"
```
Log into Azure, create certificate and Service Principal:
```
docker run --rm -it --env kv="$kv" -v $(pwd):/code mcr.microsoft.com/azure-cli /bin/bash
```

*NOTE: Run the following commands inside the az-cli Docker container you just started.*

Log into Azure:
```
az login
```
Follow the instructions in the terminal to complete login.

Create resource group:
```
az group create --name "ksscd-demo" --location "westeurope"
```

Create Key Vault:
```
az keyvault create --name "$kv" --resource-group "ksscd-demo" --location "westeurope"
```

Create test certificate, with the settings from the `certificate-policy.json` file:
```
cd /code

az keyvault certificate create --name company-certificate --vault-name "$kv" --policy "@certificate-policy.json"
```

Create Service Principal for accessing the Key Vault. **Note the values returned by the command, as they will be needed later**:
```
az ad sp create-for-rbac --name ksscd-service-principal
```

Grant the new Service Principal permissions on the Key Vault:
```
az keyvault set-policy -n "$kv" \
  --secret-permissions get \
  --key-permissions get \
  --certificate-permissions get \
  --spn "[SERVICE_PRINCIPAL_APPID]"
```

You may exit out of the Azure CLI container now, but don't close the terminal just yet.

## Minikube setup
*NOTE: Run these commands in your regular terminal.*

Start a new Minikube cluster:
```
minikube start
```

### Non-TLS version:
Install this Helm chart:
```
helm upgrade -i ksscd-demo ./ksscd-demo/ --set tls="false"
```

### TLS version
Add KSSCD repo to Helm, and install it in the cluster:
```
helm repo add csi-secrets-store-provider-azure https://azure.github.io/secrets-store-csi-driver-provider-azure/charts

helm install csi csi-secrets-store-provider-azure/csi-secrets-store-provider-azure --namespace kube-system
```

Install this Helm chart:
```
helm upgrade -i ksscd-demo ./ksscd-demo/ \
--set tls="true" \
--set keyvault.name="$kv" \
--set keyvault.tenant_id=[SERVICE_PRINCIPAL_TENANT] \
--set keyvault.credentials.id=[SERVICE_PRINCIPAL_APPID] \
--set keyvault.credentials.secret=[SERVICE_PRINCIPAL_PASSWORD]
```

# Verifying
The `tls-client` pods can be checked to verify that the HTTPS connection is working:
```
kubectl logs tls-client -n tls-client
```

# Cleanup
Destroy the `minikube` cluster:
```
minikube delete
```

Delete the created Resource Group and Service Principal in the Azure Portal, or by logging into a new `az cli` terminal:
```
docker run --rm -it mcr.microsoft.com/azure-cli /bin/bash
```
In the newly opened session:
```
az login
az group delete -n "ksscd-demo"
az ad sp delete --id [SERVICE_PRINCIPAL_APPID]
```
