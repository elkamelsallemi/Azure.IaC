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
## Use parameter files at deployment time
```bash
az deployment group create --template-file main.bicep --parameters main.parameters.json
```

## Create a key vault and secrets
To create the **keyVaultName**, **login**, and **password** variables, run each command separately. <br>
Then you can run the block of commands to create the key vault and secrets.

```bash
keyVaultName='YOUR-KEY-VAULT-NAME'
read -s -p "Enter the login name: " login
read -s -p "Enter the password: " password

az keyvault create --name $keyVaultName --location westus3 --enabled-for-template-deployment true
az keyvault secret set --vault-name $keyVaultName --name "sqlServerAdministratorLogin" --value $login --output none
az keyvault secret set --vault-name $keyVaultName --name "sqlServerAdministratorPassword" --value $password --output none
```
You're setting the **--enabled-for-template-deployment** setting on the vault so that Azure can use the secrets from your vault during deployments. If you don't set this setting then, by default, your deployments can't access secrets in your vault.

## Get the key vault's resource ID
```bash
az keyvault show --name $keyVaultName --query id --output tsv
```
## Deploy the Bicep template with parameter file
```bash
az deployment group create --template-file main.bicep --parameters main.parameters.dev.json
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
    name: appServicePlanSku.name
    tier: appServicePlanSku.tier
    capacity: appServicePlanSku.capacity
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


##Define secure parameters
The **@secure** decorator can be applied to string and object parameters that might contain secret values. When you define a parameter as **@secure**, Azure won't make the parameter values available in the deployment logs. Also, if you create the deployment interactively by using the Azure CLI or Azure PowerShell and you need to enter the values during the deployment, the terminal won't display the text on your screen.

As part of the HR application migration, you need to deploy an Azure SQL logical server and database. You'll provision the logical server with an administrator login and password. Because they're sensitive, you need these values to be secured. Here's an example declaration to create two string parameters for the SQL server's administrator details:

```bicep
@secure()
param sqlServerAdministratorLogin string

@secure()
param sqlServerAdministratorPassword string
```
## Integrate with Azure Key Vault

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2022-07-01' existing = {
  name: keyVaultName
}
```
## Use Key Vault with modules
Modules enable you to create reusable Bicep files that encapsulate a set of resources. It's common to use modules to deploy parts of your solution. Modules may have parameters that accept secret values, and you can use Bicep's Key Vault integration to provide these values securely. 

>Here's an example Bicep file that deploys a module and provides the value of the ApiKey secret parameter by taking it directly from Key Vault:

```bicep
module applicationModule 'application.bicep' = {
  name: 'application-module'
  params: {
    apiKey: keyVault.getSecret('ApiKey')
  }
}
```

Notice that in this Bicep file, the Key Vault resource is referenced by using the existing keyword. The keyword tells Bicep that the Key Vault already exists, and this code is a reference to that vault. Bicep won't redeploy it. Also, notice that the module's code uses the **getSecret()** function in the value for the module's apiKey parameter. This is a special Bicep function that can only be used with secure module parameters. Internally, Bicep translates this expression to the same kind of Key Vault reference you learned about earlier.

## Add a key vault reference to a parameter file

```json
    "sqlServerAdministratorLogin": {
      "reference": {
        "keyVault": {
          "id": "YOUR-KEY-VAULT-RESOURCE-ID"
        },
        "secretName": "sqlServerAdministratorLogin"
      }
    },
    "sqlServerAdministratorPassword": {
      "reference": {
        "keyVault": {
          "id": "YOUR-KEY-VAULT-RESOURCE-ID"
        },
        "secretName": "sqlServerAdministratorPassword"
      }
    }
```
## Use basic conditions

When you deploy a resource in Bicep, you can provide the if keyword followed by a condition. The condition should resolve to a Boolean (true or false) value. If the value is true, the resource is deployed. If the value is false, the resource is not deployed.

It's common to create conditions based on the values of parameters that you provide. For example, the following code deploys a storage account only when the deployStorageAccount parameter is set to true:

```bicep
param deployStorageAccount bool

resource storageAccount 'Microsoft.Storage/storageAccounts@2021-09-01' = if (deployStorageAccount) {
  name: 'teddybearstorage'
  location: resourceGroup().location
  kind: 'StorageV2'
  // ...
}
```
Notice that the if keyword is on the same line as the resource definition.

## Use expressions as conditions
The preceding example was quite basic. The deployStorageAccount parameter was of type bool, so it's clear whether it has a value of true or false.

In Bicep, conditions can also include expressions. In the following example, the code deploys a SQL auditing resource only when the environmentName parameter value is equal to Production:

```bicep
@allowed([
  'Development'
  'Production'
])
param environmentName string

resource auditingSettings 'Microsoft.Sql/servers/auditingSettings@2021-11-01-preview' = if (environmentName == 'Production') {
  parent: server
  name: 'default'
  properties: {
  }
}
```
```bicep
@allowed([
  'Development'
  'Production'
])
param environmentName string

var auditingEnabled = environmentName == 'Production'

resource auditingSettings 'Microsoft.Sql/servers/auditingSettings@2021-11-01-preview' = if (auditingEnabled) {
  parent: server
  name: 'default'
  properties: {
  }
}
```
## Depend on conditionally deployed resources
Notice that this Bicep code uses the question mark (?) operator within the storageEndpoint and storageAccountAccessKey properties. When the Bicep code is deployed to a production environment, the expressions are evaluated to the details from the storage account. When the code is deployed to a non-production environment, the expressions evaluate to an empty string ('').

You might wonder why this code is necessary, because auditingSettings and auditStorageAccount both have the same condition, and so you'll never need to deploy a SQL auditing settings resource without a storage account. Although this is true, Azure Resource Manager evaluates the property expressions before the conditionals on the resources. That means that if the Bicep code doesn't have this expression, the deployment will fail with a ResourceNotFound error.

```bicep
@allowed([
  'Development'
  'Production'
])
param environmentName string
param location string = resourceGroup().location
param auditStorageAccountName string = 'bearaudit${uniqueString(resourceGroup().id)}'

var auditingEnabled = environmentName == 'Production'
var storageAccountSkuName = 'Standard_LRS'

resource auditStorageAccount 'Microsoft.Storage/storageAccounts@2021-09-01' = if (auditingEnabled) {
  name: auditStorageAccountName
  location: location
  sku: {
    name: storageAccountSkuName
  }
  kind: 'StorageV2'
}

resource auditingSettings 'Microsoft.Sql/servers/auditingSettings@2021-11-01-preview' = if (auditingEnabled) {
  parent: server
  name: 'default'
  properties: {
    state: 'Enabled'
    storageEndpoint: environmentName == 'Production' ? auditStorageAccount.properties.primaryEndpoints.blob : ''
    storageAccountAccessKey: environmentName == 'Production' ? listKeys(auditStorageAccount.id, auditStorageAccount.apiVersion).keys[0].value : ''
  }
}
```
## Use copy loops

When you define a resource or a module in a Bicep template, you can use the for keyword to create a loop. Place the for keyword in the resource declaration, then specify how you want Bicep to identify each item in the loop. Typically, you loop over an array of objects to create multiple instances of a resource. The following example deploys multiple storage accounts, and their names are specified as parameter values:

```bicep
param storageAccountNames array = [
  'saauditus'
  'saauditeurope'
  'saauditapac'
]

resource storageAccountResources 'Microsoft.Storage/storageAccounts@2021-09-01' = [for storageAccountName in storageAccountNames: {
  name: storageAccountName
  location: resourceGroup().location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
  }
}]
```

In this example, the loop iterates through each item in the storageAccountNames array. Each time Bicep goes through the loop, it puts the current value into a special variable called storageAccountName, and it's used as the value of the name property. Notice that Bicep requires you put an opening bracket ([) character before the for keyword, and a closing bracket (]) character after the resource definition.

If you deployed this Bicep file, you'd see that three storage accounts were created, with their names specified by the corresponding items in the storageAccountNames array.

## Loop based on a count
You might sometimes need to loop to create a specific number of resources, and not use an array as the source. Bicep provides the range() function, which creates an array of numbers. For example, if you need to create four storage accounts called sa1 through sa4, you could use a resource definition like this:

```bicep
resource storageAccountResources 'Microsoft.Storage/storageAccounts@2021-09-01' = [for i in range(1,4): {
  name: 'sa${i}'
  location: resourceGroup().location
  kind: 'StorageV2'
  sku: {
    name: 'Standard_LRS'
  }
}]
```
When you use the range() function, you specify its start value and the number of values you want to create. For example, if you want to create storage accounts with the names sa0, sa1, and sa2, you'd use the function range(0,3).
## Access the iteration index

With Bicep, you can iterate through arrays and retrieve the index of the current element in the array. For example, let's say you want to create a logical server in each location that's specified by an array, and you want the names of the servers to be sqlserver-1, sqlserver-2, and so on. You could achieve this by using the following Bicep code:

```bicep
param locations array = [
  'westeurope'
  'eastus2'
  'eastasia'
]

resource sqlServers 'Microsoft.Sql/servers@2021-11-01-preview' = [for (location, i) in locations: {
  name: 'sqlserver-${i+1}'
  location: location
  properties: {
    administratorLogin: administratorLogin
    administratorLoginPassword: administratorLoginPassword
  }
}]

```
Notice that the name property includes the expression i+1. The first value of the i index variable is zero, so you need to add +1 to it if you want your server names to start with 1.


## Filter items with loops
In some situations, you might want to deploy resources by using copy loops combined with conditions. You can do this by combining the if and for keywords.

In the following example, the code uses an array parameter to define a set of logical servers. A condition is used with the copy loop to deploy the servers only when the environmentName property of the loop object equals Production:

```bicep
param sqlServerDetails array = [
  {
    name: 'sqlserver-we'
    location: 'westeurope'
    environmentName: 'Production'
  }
  {
    name: 'sqlserver-eus2'
    location: 'eastus2'
    environmentName: 'Development'
  }
  {
    name: 'sqlserver-eas'
    location: 'eastasia'
    environmentName: 'Production'
  }
]

resource sqlServers 'Microsoft.Sql/servers@2021-11-01-preview' = [for sqlServer in sqlServerDetails: if (sqlServer.environmentName == 'Production') {
  name: sqlServer.name
  location: sqlServer.location
  properties: {
    administratorLogin: administratorLogin
    administratorLoginPassword: administratorLoginPassword
  }
  tags: {
    environment: sqlServer.environmentName
  }
}]
```
If you deployed the preceding example, you'd see two logical servers, sqlserver-we and sqlserver-eas, but not sqlserver-eus2, because that object's environmentName property doesn't match Production.

## Control loop execution
By default, Azure Resource Manager creates resources from loops in parallel and in a non-deterministic order. When you created loops in the previous exercises, both of the Azure SQL logical servers were created at the same time. This helps to reduce the overall deployment time, because all of the resources within the loop are deployed at once.

In some cases, however, you might need to deploy resources in loops sequentially instead of in parallel, or deploy small batches of changes together in parallel. For example, if you have lots of Azure App Service apps in your production environment, you might want to deploy changes to only a small number at a time to prevent the updates from restarting all of them at the same time.

You can control the way your copy loops run in Bicep by using the @batchSize decorator. Put the decorator on the resource or module declaration with the for keyword.

Let's look at an example Bicep definition for a set of App Service applications without the @batchSize decorator:

```bicep
resource appServiceApp 'Microsoft.Web/sites@2021-03-01' = [for i in range(1,3): {
  name: 'app${i}'
  // ...
}]
```
![Screenshot](./img/1.png)









