# Connecting ADLS GEN1 with Eventhub for logging purposes

## EventHub
First, and Eventhub should be created. This will later be referenced by the ADLS GEN1 account.

### Create a new Eventhub
````console
az eventhubs eventhub create --resource-group <existing rg-name> --namespace-name <namespace name> --name <eventhub name> --message-retention 4 --partition-count 15
````

* --message-retention: Number of days to retain events for this Event Hub, value should be 1 to 7 days.

* --partition-count: Number of partitions created for the Event Hub, value should be 2 to 32.

## ADLS Gen1 

### Create the Data Lake Storage Gen1 account.
````console
az dls account create --account <adls name> --resource-group <existing rg-name>
````

### Create folders
````console
az dls fs create --account <adls name> --path /<folder name> --folder
````

#### Note: The --folder parameter ensures that the command creates a folder. If this parameter is not present, the command creates an empty file called mynewfolder at the root of the Data Lake Storage Gen1 account.

### Upload data
````console
az dls fs upload --account <adls name> --source-path "<path to data>" --destination-path "/<folder name>/<file name>"
````

### list files
````console
az dls fs list --account <adls name> --path /<folder name>
````

### rename files
````console
az dls fs move --account <adls name> --source-path /<folder name>/<file name> --destination-path /<folder name>/<new file name>
````

## Azure Monitor
Azure monitor should be setup to allow ADLS to export logs to the Eventhub. 

* Ref: https://docs.microsoft.com/en-us/cli/azure/monitor/diagnostic-settings?view=azure-cli-latest
* Ref: http://techgenix.com/azure-diagnostic-settings/
* Ref: https://docs.microsoft.com/en-us/azure/azure-monitor/platform/diagnostic-logs-overview

````
az monitor diagnostic-settings create -n DiagEventHub --resource '/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/adls-test2/providers/Microsoft.DataLakeStore/accounts/pelithneadlstest' --event-hub-rule '/subscriptions/6f66105f-d352-482f-970b-a1d2a478fb64/resourceGroups/adls-test2/providers/Microsoft.EventHub/namespaces/pelithnehub/authorizationrules/RootManageSharedAccessKey' --event-hub pelithnehub --logs '[{"category":"Audit","Enabled":true}]' --metrics '[{"category":"AllMetrics","Enabled":true}]'
````
