---
title: "Getting Started with ARM Templates"
datePublished: Mon Jul 24 2023 01:38:45 GMT+0000 (Coordinated Universal Time)
cuid: clkg78xl6000309kyaysjf074
slug: getting-started-with-arm-templates
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1690130500990/873a3ff3-e427-44df-8c5d-0464aecfa5b0.jpeg
tags: azure, arm-template

---

Infrastructure as Code (IaC) is a modern approach to managing and provisioning cloud resources, enabling developers and operations teams to automate and streamline their infrastructure deployment and configuration. [Azure Resource Manager (ARM) templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) play a crucial role in implementing IaC in Azure environments, providing a declarative syntax to define, configure, and manage Azure resources consistently and predictably. With ARM templates, we can simplify infrastructure management, reduce human error, and ensure that your Azure projects are scalable and maintainable. In this guide, we will examine the essential elements of ARM templates by addressing a straightforward task: provisioning an Azure Storage Account.

## Pre-requisites

* Ensure you have an [**Azure Account**](https://azure.microsoft.com/en-us/free/)
    
* Install [**Azure CLI**](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
    

## ARM Template Anatomy

Here is an empty ARM template:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {  },
  "variables": {  },
  "functions": [  ],
  "resources": [  ],
  "outputs": {  }
}
```

The sections below will not have a detailed explanation:

* `$schema`: Location of the JSON schema file that describes the version of the template language.
    
* `contentVersion`: Version of the template. We can provide any value for this element.
    

### Resources

This section defines the [resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/resource-declaration) that are deployed or updated. Create a file named `template.json` with the following content:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {  },
  "variables": {  },
  "functions": [  ],
  "resources":  [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2019-04-01",
      "name": "myuniquestorage001",
      "location": "[resourceGroup().location]",
      "dependsOn": [],
      "tags": {},
      "sku": {
        "name": "Standard_GRS"
      },
      "kind": "StorageV2",
      "properties": {}
    }
  ],
  "outputs": {  }
}
```

Most resources possess a set of common properties, such as:

* `Name`: Name of the resource.
    
* `Type`: The [type of resource](https://learn.microsoft.com/en-us/azure/templates/) to deploy. This value is a combination of the namespace of the resource provider and the resource type (separated by a `/`).
    
* `ApiVersion`: The version of the REST API used for creating the resource, which determines the available properties.
    
* `Tags`: Tags that are associated with the resource.
    
* `Location`: Supported geographical locations for the provided resource. We typically deploy resources to the same resource group in which the deployment is created.
    
* `DependsOn`: ARM templates need the manual creation of resource dependencies. These dependencies dictate the sequence in which Azure deploys the resources.
    
* `Properties`: Resource-specific configuration settings.
    
* `Kind`: Some resources allow a value that defines the type of resource you deploy.
    
* `SKU`: Some resources allow values that define the SKU to deploy.
    

To deploy the template, a resource group is required:

```powershell
az group create -l eastus -n MyResourceGroup
```

After creating the resource group, execute the following command:

```powershell
az deployment group create --resource-group MyResourceGroup --template-file .\template.json
```

Navigate to the created resource group in the Azure portal to view the deployment execution:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1690156809424/77d26fa0-3d2a-4abb-ae3f-40ad9a28b268.png align="center")

### Parameters

[Parameters](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/parameters) enable you to provide various values to the ARM template during the deployment:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {     
    "storageAccountName": {
      "type": "string",
      "metadata": {
          "description": "The name of the storage account."
      }
    } 
  },
  "variables": {  },
  "functions": [  ],
  "resources":  [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2019-04-01",
      "name": "[parameters('storageAccountName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": [],
      "tags": {},
      "sku": {
        "name": "Standard_GRS"
      },
      "kind": "StorageV2",
      "properties": {}
    }
  ],
  "outputs": {  }
}
```

To pass inline parameters, you can execute a command like this:

```powershell
az deployment group create --resource-group MyResourceGroup --template-file .\template.json --parameters storageAccountName='myuniquestorage002'
```

Instead of using inline values for parameters in your script, you can utilize a [JSON file](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/parameter-files) containing the parameter values. Create a `parameter.json` file with the following content:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "storageAccountName": {
      "value": "myuniquestorage003"
    }
  }
}
```

To pass a local parameter file, use `@` to specify a local file:

```powershell
az deployment group create --resource-group MyResourceGroup --template-file .\template.json --parameters '@parameters.json'
```

### Variables

Like parameters, [variables](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/variables) can contain values but also template expressions. They are often used to simplify templates by minimizing the usage of template expressions. Let's modify our script to automatically generate a name based on a prefix:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {     
    "storageAccountPrefix": {
      "type": "string",
      "metadata": {
          "description": "The prefix of the storage account."
      }
    } 
  },
  "variables": { 
    "storageAccountName": "[concat(toLower(parameters('storageAccountPrefix')), uniqueString(resourceGroup().id))]"
   },
  "functions": [  ],
  "resources":  [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2019-04-01",
      "name": "[variables('storageAccountName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": [],
      "tags": {},
      "sku": {
        "name": "Standard_GRS"
      },
      "kind": "StorageV2",
      "properties": {}
    }
  ],
  "outputs": {  }
}
```

We can utilize all the functions mentioned [here](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions). Execute the following command:

```powershell
az deployment group create --resource-group MyResourceGroup --template-file .\template.json --parameters storageAccountPrefix='mystorage'
```

### Outputs

The [outputs](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/outputs?tabs=azure-powershell) section defines values and information returned from the deployment. Update the template as follow:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {     
    "storageAccountPrefix": {
      "type": "string",
      "metadata": {
          "description": "The prefix of the storage account."
      }
    } 
  },
  "variables": { 
    "storageAccountName": "[concat(toLower(parameters('storageAccountPrefix')), uniqueString(resourceGroup().id))]"
   },
  "functions": [  ],
  "resources":  [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2019-04-01",
      "name": "[variables('storageAccountName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": [],
      "tags": {},
      "sku": {
        "name": "Standard_GRS"
      },
      "kind": "StorageV2",
      "properties": {}
    }
  ],
  "outputs": {  
      "endpoints": {
      "type": "object",
      "value": "[reference(variables('storageAccountName')).primaryEndpoints]"
    }
  }
}
```

Deploy the template, and you should see an output similar to this:

```json
... 
"outputs": {
      "endpoints": {
        "type": "Object",
        "value": {
          "blob": "https://mystorageeksid6zaqw34i.blob.core.windows.net/",
          "dfs": "https://mystorageeksid6zaqw34i.dfs.core.windows.net/",
          "file": "https://mystorageeksid6zaqw34i.file.core.windows.net/",
          "queue": "https://mystorageeksid6zaqw34i.queue.core.windows.net/",
          "table": "https://mystorageeksid6zaqw34i.table.core.windows.net/",
          "web": "https://mystorageeksid6zaqw34i.z13.web.core.windows.net/"
        }
      }
    },
...
```

### Functions

[Functions](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/user-defined-functions) enable you to create complex expressions that we would prefer not to repeat throughout the template. Let's define a function to generate unique names based on a prefix:

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {     
    "storageAccountPrefix": {
      "type": "string",
      "metadata": {
          "description": "The prefix of the storage account."
      }
    } 
  },
  "variables": { 
    "storageAccountName": "[mynamespase.uniqueName(parameters('storageAccountPrefix'))]"
   },
  "functions": [ 
   {
    "namespace": "mynamespase",
    "members": {
      "uniqueName": {
        "parameters": [
          {
            "name": "namePrefix",
            "type": "string"
          }
        ],
        "output": {
          "type": "string",
          "value": "[concat(toLower(parameters('namePrefix')), uniqueString(resourceGroup().id))]"
        }
      }
    }
  }
   ],
  "resources":  [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2019-04-01",
      "name": "[variables('storageAccountName')]",
      "location": "[resourceGroup().location]",
      "dependsOn": [],
      "tags": {},
      "sku": {
        "name": "Standard_GRS"
      },
      "kind": "StorageV2",
      "properties": {}
    }
  ],
  "outputs": {  
      "endpoints": {
      "type": "object",
      "value": "[reference(variables('storageAccountName')).primaryEndpoints]"
    }
  }
}
```

> When defining a user function, there are some restrictions:
> 
> * The function can't access variables.
>     
> * The function can only use parameters that are defined in the function. When you use the [parameters](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-deployment#parameters) function within a user-defined function, you're restricted to the parameters for that function.
>     
> * The function can't call other user-defined functions.
>     
> * The function can't use the [reference](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#reference) function or any of the [list](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#list) functions.
>     
> * Parameters for the function can't have default values.
>     

## Deployment Modes

When deploying your resources, we can specify whether the deployment should be an incremental update or a complete update:

* `complete`: Resource Manager deletes resources that exist in the resource group but aren't specified in the template.
    
* `incremental`: Resource Manager leaves unchanged resources that exist in the resource group but aren't specified in the template.
    

The incremental mode is the default setting. To use the complete mode, execute a command like:

```powershell
az deployment group create --resource-group MyResourceGroup --template-file .\template.json --parameters storageAccountPrefix='mystorage' --mode complete
```

## Preview

To preview the changes that will occur, Azure Resource Manager offers the `what-if` operation:

```powershell
az deployment group what-if --resource-group MyResourceGroup --template-file .\template.json --parameters storageAccountPrefix='mystorage
```

We can use `--confirm-with-what-if` to preview the changes and get prompted to continue with the deployment:

```powershell
 az deployment group create --confirm-with-what-if --resource-group MyResourceGroup --template-file .\template.json --parameters storageAccountPrefix='mystorage'
```

In conclusion, ARM templates are a powerful tool for implementing Infrastructure as Code in Azure environments. By understanding the essential elements of ARM templates, such as resources, parameters, variables, outputs, and functions, we can effectively manage your Azure resources and enhance your cloud projects. Thanks, and happy coding.