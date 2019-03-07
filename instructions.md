# Connecting ADLS GEN1 with Eventhub for logging purposes

## EventHub
First, and Eventhub should be created. This will later be referenced by the ADLS GEN1 account.

### Create Event Hubs namespace

````
az eventhubs namespace create --name <Event Hubs namespace> --resource-group <resource group name> -l <region>

````

This will give an output similar to this
````
{
  "createdAt": "2019-03-06T18:24:45.767000+00:00",
  "id": "/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/adls-test2/providers/Microsoft.EventHub/namespaces/pelithnehub2ns",
  "isAutoInflateEnabled": false,
  "location": "West Europe",
  "maximumThroughputUnits": 0,
  "metricId": "6f66105f-d352-482f-970b-a1d2a478fb64:pelithnehub2ns",
  "name": "pelithnehub2ns",
  "provisioningState": "Succeeded",
  "resourceGroup": "adls-test2",
  "serviceBusEndpoint": "https://pelithnehub2ns.servicebus.windows.net:443/",
  "sku": {
    "capacity": 1,
    "name": "Standard",
    "tier": "Standard"
  },
  "tags": {},
  "type": "Microsoft.EventHub/Namespaces",
  "updatedAt": "2019-03-06T18:25:12.613000+00:00"
}
````

### Create a new Eventhub
````console
az eventhubs eventhub create --resource-group <existing rg-name> --namespace-name <namespace name> --name <eventhub name> --message-retention 4 --partition-count 15
````


* --message-retention: Number of days to retain events for this Event Hub, value should be 1 to 7 days.

* --partition-count: Number of partitions created for the Event Hub, value should be 2 to 32.

You should get a response similar to this:

````

  "captureDescription": null,
  "createdAt": "2019-03-06T18:28:57.880000+00:00",
  "id": "/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/adls-test2/providers/Microsoft.EventHub/namespaces/pelithnehub2ns/eventhubs/pelithnehub2",
  "location": "West Europe",
  "messageRetentionInDays": 4,
  "name": "pelithnehub2",
  "partitionCount": 15,
  "partitionIds": [
    "0",
    "1",
    "2",
    "3",
    "4",
    "5",
    "6",
    "7",
    "8",
    "9",
    "10",
    "11",
    "12",
    "13",
    "14"
  ],
  "resourceGroup": "adls-test2",
  "status": "Active",
  "type": "Microsoft.EventHub/Namespaces/EventHubs",
  "updatedAt": "2019-03-06T18:28:58.210000+00:00"
}
````


## ADLS Gen1 

### Create the Data Lake Storage Gen1 account.
````console
az dls account create --account <adls name> --resource-group <existing rg-name>
````

You should get a response similar to this. Make a note of the resource id ("id"), as this will be used in a later step.
````
{
  "accountId": "8e1db8cf-388a-4812-a257-196b9994433b",
  "creationTime": "2019-03-06T18:30:48.685647+00:00",
  "currentTier": "Consumption",
  "defaultGroup": null,
  "encryptionConfig": {
    "keyVaultMetaInfo": null,
    "type": "ServiceManaged"
  },
  "encryptionProvisioningState": null,
  "encryptionState": "Enabled",
  "endpoint": "pelithneadls.azuredatalakestore.net",
  "firewallAllowAzureIps": "Disabled",
  "firewallRules": [],
  "firewallState": "Disabled",
  "id": "/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/adls-test2/providers/Microsoft.DataLakeStore/accounts/pelithneadls",
  "identity": {
    "principalId": "63918483-413b-4a56-9166-f2d90ab82c40",
    "tenantId": "72f988bf-86f1-41af-91ab-2d7cd011db47"
  },
  "lastModifiedTime": "2019-03-06T18:30:48.685647+00:00",
  "location": "westeurope",
  "name": "pelithneadls",
  "newTier": "Consumption",
  "provisioningState": "Succeeded",
  "resourceGroup": "adls-test2",
  "state": "Active",
  "tags": null,
  "trustedIdProviderState": "Disabled",
  "trustedIdProviders": [],
  "type": "Microsoft.DataLakeStore/accounts",
  "virtualNetworkRules": []
}
````



## Azure Monitor
Azure monitor should be setup to allow ADLS to export logs to the Eventhub. 

* Ref: https://docs.microsoft.com/en-us/cli/azure/monitor/diagnostic-settings?view=azure-cli-latest
* Ref: http://techgenix.com/azure-diagnostic-settings/
* Ref: https://docs.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-logs-overview


The example below creates a diagnistic setting in the specified ADLS, and activates some logs and metrics:
````
az monitor diagnostic-settings create -n DiagEventHub --resource '/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/adls-test2/providers/Microsoft.DataLakeStore/accounts/pelithadls' --event-hub-rule '/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/adls-test2/providers/Microsoft.EventHub/namespaces/pelithubns/authorizationrules/RootManageSharedAccessKey' --event-hub pelithub --logs '[{"category":"Audit","Enabled":true}]' --metrics '[{"category":"AllMetrics","Enabled":true}]'
````

## Testing the solution
The activities below should create events in ADLS, which should be exported to the Eventhub, so that they can be consumed.

### Generate events
Creating a folder in ADLS will generate an audit event:

````console
az dls fs create --account <adls name> --path /<folder name> --folder
````


### Consume events
One way to consume events from the event hub, is to use the "Azure Event Hub Explorer" extension to VS Code. It can be found here: https://marketplace.visualstudio.com/items?itemName=Summer.azure-event-hub-explorer

Follow instructions to start monitoring event hub messages. 

Event example
````
{
      "time": "2019-03-07T08:13:23.870Z",
      "resourceId": "/SUBSCRIPTIONS/6F66105F-D352-482F-970B-A1D2A478FB64/RESOURCEGROUPS/ADLS-TEST2/PROVIDERS/MICROSOFT.DATALAKESTORE/ACCOUNTS/PELITHADLS",
      "category": "Audit",
      "operationName": "CreateDirectory",
      "resultType": "Success",
      "resultSignature": "0",
      "identity": "pelithne@microsoft.com",
      "properties": {
        "StreamName": "adl://pelithadls.azuredatalakestore.net/testfolder2",
        "UserId": "D958C518-E765-4F1A-ADFF-5AB347FBC21C"
      }
    },
````





