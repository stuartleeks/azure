# Getting Started with StorageOS on Azure with acs-engine

Follow the steps below to set up a kubernetes cluster in [Azure](https://azure.microsoft.com) using [acs-engine](https://github.com/azure/acs-engine) and install [StorageOS](https://storageos.com/).

## Get an Azure Subscription

For the steps below you will need an [Azure Subscription](https://azure.com/free).

To deploy a cluster through acs-engine you need your Azure Subscription ID. 
The [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) is a great way to get the ID of your subscription. If you don't have the Azure CLI installed then you can sign in to [Azure Cloud Shell](https://shell.azure.com) to run the commands there:


```bash
# save the current subscription id in a variable for later...
subscriptionId=$(az account show --output tsv --query id)

# or show the current account (to copy locally if running from the Cloud Shell)
az account show --output json

# or list your subscriptions (if you have multiple subscriptions)
az account list
``` 

## Create a kubernetes cluster


Grab the [latest release of acs-engine](https://github.com/Azure/acs-engine/releases). 

There are various [examples of kubernetes cluster definitions](https://github.com/Azure/acs-engine/tree/master/examples) for acs-engine, but the important thing is to specify kubernetes 1.10 as StorageOS requires this version to support running with kubelet in a container.

Example definition (note the `orchestratorRelease` property):

```json
{
  "apiVersion": "vlabs",
  "properties": {
    "orchestratorProfile": {
      "orchestratorType": "Kubernetes",
      "orchestratorRelease": "1.10"
    },
    "masterProfile": {
      "count": 1,
      "dnsPrefix": "",
      "vmSize": "Standard_D2_v2"
    },
    "agentPoolProfiles": [
      {
        "name": "agentpool1",
        "count": 3,
        "vmSize": "Standard_D2_v2",
        "availabilityProfile": "AvailabilitySet"
      }
    ],
    "linuxProfile": {
      "adminUsername": "azureuser",
      "ssh": {
        "publicKeys": [
          {
            "keyData": ""
          }
        ]
      }
    },
    "servicePrincipalProfile": {
      "clientId": "",
      "secret": ""
    }
  }
}
```

With this config file (saved as `kubernetes.1.10.json` for this example) and the `$subscriptionId` set from the previous section you can run `acs-engine deploy`:

```bash
acs-engine deploy --api-model kubernetes.1.10.json  --subscription-id $subscriptionid --resource-group storageosazure  --location northeurope --dns-prefix storageosazure --auto-suffix
```

This will deploy your kubernetes cluster to the region of your choice. For more information on `acs-engine deploy` see [the documentation](https://github.com/Azure/acs-engine/blob/master/docs/kubernetes/deploy.md).

The deployment also creates a KUBECONFIG for your cluster under the `_output` directory. The exact filename depends on the dns prefix and location you specified. To set the KUBECONFIG for the example above:

```bash
KUBECONFIG=~/mydir/_output/storageosazure/kubeconfig/kubeconfig.northeurope.json
```

Running `kubectl cluster-version` should now connect to your new cluster!


## Deploying StorageOS

To deploy StorageOS you will need
* helm - see [instructions for installation](https://github.com/kubernetes/helm#install)
* the StorageOS helm chart - clone it from https://github.com/storageos/helm-chart


The StorageOS helm chart needs to know the name or IP Address of one of the agent nodes in the cluster

```bash
# From the StorageOS helm-chart directory
helm install . --name storageos --set cluster.join=$(kubectl get node -l kubernetes.io/role==agent --output jsonpath={.items[0].metadata.name})

# NOTE: Check the output for the helm chart and run the commands as instructed
```
