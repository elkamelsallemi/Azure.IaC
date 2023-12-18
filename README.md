# AzureInfrastructure
Build reusable Bicep templates for Azure Infrastructure by using parameters 


## Azure pipelines deployment Status :
[![Build Status](https://dev.azure.com/sallemi-elkamel/CodeHub/_apis/build/status%2FCodeHub.Azure.IaC?branchName=main)](https://dev.azure.com/sallemi-elkamel/CodeHub/_build/latest?definitionId=19&branchName=main)


## Bicep deployment task to the pipeline
Example Project Path : <br>
├─**Azure.IaC** <br>
│   .gitignore <br>
│   LICENSE <br>
│   README.md <br>
│
├───**deploy** <br>
│       **azure-pipelines.yml** <br>
│       **main.bicep** <br>
│
└───templates <br>
    │   README.md <br>
    │ <br>
    ├───AzureSQL-VNet <br>

```yml
- task: AzureResourceManagerTemplateDeployment@3
  inputs:
    connectedServiceName: 'MyServiceConnection'
    location: 'westus3'
    resourceGroupName: Example
    csmFile: deploy/main.bicep
    overrideParameters: >
        -parameterName parameterValue
```
The first line specifies **AzureResourceManagerTemplateDeployment@3**. It tells Azure Pipelines that the task you want to use for this step is named **AzureResourceManagerTemplateDeployment**, and you want to use version **3** of the task.

When you use the Azure Resource Group Deployment task, you specify **inputs** to tell the task what to do. Here are some **inputs** you might specify when you use the task:

**connectedServiceName** is the name of the service connection to use.
**location** needs to be specified even though its value might not be used. The Azure Resource Group Deployment task can also create a resource group for you, and if it does, it needs to know the Azure region in which to create the resource group. In this module, you'll specify the **location** input value but its value isn't used.
**resourceGroupName** specifies the name of the resource group that the Bicep file should be deployed to.
**overrideParameters** contains any parameter values you want to pass into your Bicep file at deployment time.

## Variable in Azure Pipeline
After you create a variable, you'll use a specific syntax to refer to the variable in your pipeline's YAML file:

```yml
- task: AzureResourceManagerTemplateDeployment@3
  inputs:
    connectedServiceName: $(ServiceConnectionName)
    location: $(DeploymentDefaultLocation)
    resourceGroupName: $(ResourceGroupName)
    csmFile: deploy/main.bicep
    overrideParameters: >
      -environmentType $(EnvironmentType)
```

The pipeline definition file format includes a special **$(VariableName)** syntax. You can refer to any variable by using this approach, whether it's secret or not.