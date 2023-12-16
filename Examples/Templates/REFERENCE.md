# Azure CLI References
## Sign in to Azure
```bash
Sign in to Azure
```
## Set the default subscription
```bash
az account set --subscription {your subscription ID}
```
## Set the default resource group
```bash
az configure --defaults group=[sandbox resource group name]
```
## Deploy template
```bash
az deployment group create --template-file main.bicep
```
# Use parameter files at deployment time
```bash
az deployment group create --template-file main.bicep --parameters main.parameters.json
```
# Bicep Refrences
## Declare a parameter : 
```bicep
param environmentName string
param environmentName string = 'dev'
```
Here's an example of a string parameter named location with a default value set to the location of the current resource group
```bicep
param location string = resourceGroup().location
```
## Parameter types :

**string**, which lets you enter arbitrary text.<br>
**int**, which lets you enter a number.<br>
**bool**, which represents a Boolean (true or false) value.<br>
**object** and array, which represent structured data and lists.<br>

# Objects
```bicep
param appServicePlanSku object = {
  name: 'F1'
  tier: 'Free'
  capacity: 1
}
```
When you reference the parameter in the template, you can select the individual properties of the object by using a dot followed by the name of the property, like in this example:
```bicep
resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: appServicePlanName
  location: location
  sku: {
    name: **appServicePlanSku.name**
    tier: **appServicePlanSku.tier**
    capacity: **appServicePlanSku.capacity**
  }
}
```

you might use an object parameter is for specifying resource tags. You can attach custom tag metadata to the resources that you deploy, which you can use to identify important information about a resource
```bicep
param resourceTags object = {
  EnvironmentName: 'Test'
  CostCenter: '1000100'
  Team: 'Human Resources'
}
```

Whenever you define a resource in your Bicep file, you can reuse it wherever you define the tags property:
```bicep
resource appServiceApp 'Microsoft.Web/sites@' = {
  name: appServiceAppName
  location: location
  tags: resourceTags
  kind: 'app'
  properties: {
    serverFarmId: appServicePlan.id
  }
}
```
## Arrays
Let's consider an example. Azure Cosmos DB lets you create database accounts that span multiple regions, and it automatically handles the data replication for you. When you deploy a new database account, you need to specify the list of Azure regions that you want the account to be deployed into. Often, you'll need to have a different list of locations for different environments. For example, to save money in your test environment, you might use only one or two locations. But in your production environment, you might use several locations.

You can create an array parameter that specifies a list of locations:
```bicep
param cosmosDBAccountLocations array = [
  {
    locationName: 'australiaeast'
  }
  {
    locationName: 'southcentralus'
  }
  {
    locationName: 'westeurope'
  }
]
```
When you declare your Azure Cosmos DB resource, you can now reference the array parameter:
```bicep
resource account 'Microsoft.DocumentDB/databaseAccounts@2022-08-15' = {
  name: accountName
  location: location
  properties: {
   locations: cosmosDBAccountLocations
  }
}
```

## Specify a list of allowed values

Sometimes you need to make sure that a parameter has certain values. For example, your team might decide that production App Service plans should be deployed by using the Premium v3 SKUs. To enforce this rule, you can use the **@allowed** parameter decorator. A parameter decorator is a way of giving Bicep information about what a parameter's value needs to be. Here's how a string parameter named appServicePlanSkuName can be restricted so that only a few specific values can be assigned:
```bicep
@allowed([
  'P1v3'
  'P2v3'
  'P3v3'
])
param appServicePlanSkuName string
```
## Restrict parameter length and values
```bicep
@minLength(5)
@maxLength(24)
param storageAccountName string
```
## Add descriptions to parameters
```bicep
@description('The locations into which this Cosmos DB account should be configured. This parameter needs to be a list of objects,each of which has a locationName property.') 
param cosmosDBAccountLocations array
```
```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2022-07-01' existing = {
  name: keyVaultName
}
```

module applicationModule 'application.bicep' = {
  name: 'application-module'
  params: {
    apiKey: keyVault.getSecret('ApiKey')
  }
}