---
title: "Exploring Modules in Azure Bicep"
datePublished: Wed Aug 09 2023 18:27:21 GMT+0000 (Coordinated Universal Time)
cuid: cll42bmib000c09mm5q3d9nxx
slug: exploring-modules-in-azure-bicep
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691541335218/4a522acd-cdeb-4ab0-b137-b6b124afdbe8.png
tags: azure, arm-template, bicep

---

In the article [Getting Started with Azure Bicep](https://blog.raulnq.com/getting-started-with-azure-bicep), we discussed the fundamental concepts required to start using it. Today, we will explore the topic of modules.

## What are modules?

Modules are reusable building blocks of code that enable us to create, manage, and deploy Azure resources in a more organized and efficient manner. A module is a regular Bicep file that enables us to bundle multiple resources, parameters, variables, and outputs into a single package, which can then be invoked from other Bicep files.

## Why use modules?

These modules help to reduce code duplication, enhance maintainability, and simplify complex infrastructure deployments by encapsulating resource configurations.

## Anatomy

The syntax for defining a [module](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/modules) is:

```json
module <symbolic-name> '<path-to-file>' = {
  name: '<linked-deployment-name>'
  params: {
    <parameter-names-and-values>
  }
}
```

> The **name** property is required. It becomes the name of the nested deployment resource in the generated template.

When creating a module, the most crucial aspect is defining the interface, the inputs it requires, and what outputs it provides:

* **Inputs**: The `params` specified in the module definition must correspond to the parameters in the Bicep file.
    
* **Outputs**: We can get values from a module and use them in the main Bicep file using the syntax `<symbolic-name>.outputs.<output-name>`.
    

## My first module

To illustrate the usage and benefits, we will refactor the following script (`main.bicep`) using modules:

```json
param prefix string = 'myapp'
param location string = resourceGroup().location
var appServicePlanName = '${toLower(prefix)}-app-${uniqueString(resourceGroup().id)}'
var webSiteName = '${toLower(prefix)}-web-${uniqueString(resourceGroup().id)}'
var cpuMetricAlertName = '${appServicePlanName}-cpu-alert'
var actionGroupName = '${toLower(prefix)}-action-group-${uniqueString(resourceGroup().id)}'

resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: appServicePlanName
  location: location
  properties: {
    reserved: true
  }
  sku: {
    name: 'B1'
    capacity: 1
  }
  kind: 'linux'
}

resource appService 'Microsoft.Web/sites@2020-06-01' = {
  name: webSiteName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|7.0'
      alwaysOn: true
    }
  }
}

resource developmentTeamActionGroup 'Microsoft.Insights/actionGroups@2022-06-01' = {
  name: actionGroupName
  location: 'Global'
  properties: {
    groupShortName: 'devteam'
    enabled: true
    emailReceivers:[
      {
        name:'developer1'
        emailAddress:'raulnq@gmail.com'
      }
    ]
  }
}

resource cpuMetricAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  location: 'Global'
  name: cpuMetricAlertName
  properties:{
    severity: 2
    enabled: true
    evaluationFrequency: 'PT1H'
    windowSize: 'PT1H'
    autoMitigate: true
    targetResourceType: 'Microsoft.Web/serverFarms'
    targetResourceRegion: location
    scopes: [
      appServicePlan.id
    ]
    criteria: {
      allOf: [
          {
              threshold: 75
              name: 'Metric1'
              metricNamespace: 'Microsoft.Web/serverFarms'
              metricName: 'CpuPercentage'
              operator: 'GreaterThan'
              timeAggregation: 'Average'
              criterionType: 'StaticThresholdCriterion'
          }
      ]
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
    }
    actions: [
      {
          actionGroupId: developmentTeamActionGroup.id
      }
    ]
  }
}
```

The script creates an App Service and an Alert associated with its CPU usage. Based on this, we will create two modules: one for the App Service and another for the Alert. Let's create an `appService.bicep` file with the following content:

```json
param appServicePlanName string
param location string
param webSiteName string

resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: appServicePlanName
  location: location
  properties: {
    reserved: true
  }
  sku: {
    name: 'B1'
    capacity: 1
  }
  kind: 'linux'
}

resource appService 'Microsoft.Web/sites@2020-06-01' = {
  name: webSiteName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|7.0'
      alwaysOn: true
    }
  }
}

output appServicePlanId string = appServicePlan.id
```

And an `alert.bicep` file as follows:

```json
param cpuMetricAlertName string
param location string
param threshold int
param appServicePlanId string
param actionGroupId string

resource cpuMetricAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  location: 'Global'
  name: cpuMetricAlertName
  properties:{
    severity: 2
    enabled: true
    evaluationFrequency: 'PT1H'
    windowSize: 'PT1H'
    autoMitigate: true
    targetResourceType: 'Microsoft.Web/serverFarms'
    targetResourceRegion: location
    scopes: [
      appServicePlanId
    ]
    criteria: {
      allOf: [
          {
              threshold: threshold
              name: 'Metric1'
              metricNamespace: 'Microsoft.Web/serverFarms'
              metricName: 'CpuPercentage'
              operator: 'GreaterThan'
              timeAggregation: 'Average'
              criterionType: 'StaticThresholdCriterion'
          }
      ]
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
    }
    actions: [
      {
          actionGroupId: actionGroupId
      }
    ]
  }
}
```

Then modify the original `main.bicep` file as follows:

```json
param prefix string = 'myapp'
param location string = resourceGroup().location
var appServicePlanName = '${toLower(prefix)}-app-${uniqueString(resourceGroup().id)}'
var webSiteName = '${toLower(prefix)}-web-${uniqueString(resourceGroup().id)}'
var cpuMetricAlertName = '${appServicePlanName}-cpu-alert'
var actionGroupName = '${toLower(prefix)}-action-group-${uniqueString(resourceGroup().id)}'

module webApp './appService.bicep' = {
  name: '${webSiteName}-deployment'
  params: {
    appServicePlanName: appServicePlanName
    location : location
    webSiteName : webSiteName
  }
}

resource developmentTeamActionGroup 'Microsoft.Insights/actionGroups@2022-06-01' = {
  name: actionGroupName
  location: 'Global'
  properties: {
    groupShortName: 'devteam'
    enabled: true
    emailReceivers:[
      {
        name:'developer1'
        emailAddress:'raulnq@gmail.com'
      }
    ]
  }
}

module cpuMetricAlert './alert.bicep' = {
  name: '${cpuMetricAlertName}-deployment'
  params: {
    appServicePlanId: webApp.outputs.appServicePlanId
    cpuMetricAlertName: cpuMetricAlertName
    location : location
    actionGroupId : developmentTeamActionGroup.id
    threshold: 75
  }
}
```

This new `main.bicep` file will generate the same resources, but it will be much easier to add a new App Service and its corresponding Alert. We hope this article helps in your Azure Bicep journey! Thanks, and happy coding.