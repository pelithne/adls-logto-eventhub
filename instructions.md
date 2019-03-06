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


Example:
````
az monitor diagnostic-settings create -n DiagEventHub --resource '/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/adls-test2/providers/Microsoft.DataLakeStore/accounts/pelithneadlstest' --event-hub-rule '/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/adls-test2/providers/Microsoft.EventHub/namespaces/pelithnehub/authorizationrules/RootManageSharedAccessKey' --event-hub pelithnehub --logs '[{"category":"Audit","Enabled":true}]' --metrics '[{"category":"AllMetrics","Enabled":true}]'
````

## Testing the solution
The activities below should create events in ADLS, which should be exported to the Eventhub, so that they can be consumed.

### Generate events

#### Create folders in ADLS
````console
az dls fs create --account <adls name> --path /<folder name> --folder
````

#### Note: The --folder parameter ensures that the command creates a folder. If this parameter is not present, the command creates an empty file called mynewfolder at the root of the Data Lake Storage Gen1 account.

#### Upload data to ADLS
````console
az dls fs upload --account <adls name> --source-path "<path to data>" --destination-path "/<folder name>/<file name>"
````

#### List files
````console
az dls fs list --account <adls name> --path /<folder name>
````

#### Rename files
````console
az dls fs move --account <adls name> --source-path /<folder name>/<file name> --destination-path /<folder name>/<new file name>
````
### Consume events
You need a 
````
az eventhubs namespace authorization-rule keys list --resource-group dummyresourcegroup --namespace-name dummynamespace --name RootManageSharedAccessKey
````
Should give output similar to this
````
{
  "aliasPrimaryConnectionString": null,
  "aliasSecondaryConnectionString": null,
  "keyName": "RootManageSharedAccessKey",
  "primaryConnectionString": "Endpoint=sb://pelithnehub2ns.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=0aSQJrmKpmLViR008rtmd+qXKasFUbj+qI2huxatYjo=",
  "primaryKey": "0aSQJrmKpmLViR008rtmd+qXKasFUbj+qI2huxatYjo=",
  "secondaryConnectionString": "Endpoint=sb://pelithnehub2ns.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=GLDe6R1G+Drcu6PeEgilOKR/y50nCJJ8RErZ9OvVEYU=",
  "secondaryKey": "GLDe6R1G+Drcu6PeEgilOKR/y50nCJJ8RErZ9OvVEYU="
}
````

#### Create a Python script to receive events
 
Create a script called recv.py.
Paste the following code into recv.py, replacing the ADDRESS, USER, and KEY values with the values you obtained from the Azure portal in the previous section:
````python
import os
import sys
import logging
import time
from azure.eventhub import EventHubClient, Receiver, Offset

logger = logging.getLogger("azure")

# Address can be in either of these formats:
# "amqps://<URL-encoded-SAS-policy>:<URL-encoded-SAS-key>@<mynamespace>.servicebus.windows.net/myeventhub"
# "amqps://<mynamespace>.servicebus.windows.net/myeventhub"
# For example:
ADDRESS = "amqps://mynamespace.servicebus.windows.net/myeventhub"

# SAS policy and key are not required if they are encoded in the URL
USER = "RootManageSharedAccessKey"
KEY = "namespaceSASKey"
CONSUMER_GROUP = "$default"
OFFSET = Offset("-1")
PARTITION = "0"

total = 0
last_sn = -1
last_offset = "-1"
client = EventHubClient(ADDRESS, debug=False, username=USER, password=KEY)
try:
    receiver = client.add_receiver(CONSUMER_GROUP, PARTITION, prefetch=5000, offset=OFFSET)
    client.run()
    start_time = time.time()
    for event_data in receiver.receive(timeout=100):
        last_offset = event_data.offset
        last_sn = event_data.sequence_number
        print("Received: {}, {}".format(last_offset, last_sn))
        total += 1

    end_time = time.time()
    client.stop()
    run_time = end_time - start_time
    print("Received {} messages in {} seconds".format(total, run_time))

except KeyboardInterrupt:
    pass
finally:
    client.stop()
````

https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-python-get-started-receive
