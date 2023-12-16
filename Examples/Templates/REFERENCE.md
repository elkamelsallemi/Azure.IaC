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