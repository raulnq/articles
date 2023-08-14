---
title: "Azure Bicep: How to Provision an API Management"
datePublished: Mon Aug 14 2023 03:50:57 GMT+0000 (Coordinated Universal Time)
cuid: cllac7tio00090aju1bzk0o6z
slug: azure-bicep-how-to-provision-an-api-management
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1691945109965/fa0e1006-5dc8-4cee-9524-35747fd950ac.png
tags: azure, api-management, bicep

---

[Azure API](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts) Management is a fully managed service that helps organizations publish, secure, transform, maintain, and monitor APIs. It provides a scalable and secure environment for API hosting, allowing developers to create, manage, and analyze APIs in a streamlined manner. While there are many benefits to using a tool like this, there are some challenges, such as keeping all our environments in sync, particularly if we are managing this task manually. To address this particular issue, we can use [Azure Bicep](https://blog.raulnq.com/getting-started-with-azure-bicep) for provisioning API management and maintaining consistency across our environments.

## Pre-requisites

* Ensure you have an [Azure Account](https://azure.microsoft.com/en-us/free/)
    
* Install [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
    
* Install [Bicep CLI](https://learn.microsoft.com/es-es/azure/azure-resource-manager/bicep/install#azure-cli) (`az bicep install`)
    

## Concepts

Before diving into the details, let's clarify some key concepts:

* **Operation**: An action performed on a resource.
    
* **API**: Represents a set of operations.
    
* **Product**: A collection of one or more APIs.
    
* **Subscription**: Permissions to access a product or API.
    
* **Policies**: Collection of statements executed sequentially on the request or response to change the behavior of an API.
    

## Creating an API Management Instance

Create a `main.bicep` file containing the following content:

```json
param apiManagementName string
param location string = resourceGroup().location

resource apiManagementInstance 'Microsoft.ApiManagement/service@2020-12-01' = {
  name: apiManagementName
  location: location
  sku:{
    capacity: 0
    name: 'Consumption'
  }
  properties:{
    virtualNetworkType: 'None'
    publisherEmail: 'raulnq@gmail.com'
    publisherName: 'MyCompany'
  }
}

resource apiManagementInstancePolicy 'Microsoft.ApiManagement/service/policies@2020-12-01' = {
  name: 'policy'
  parent: apiManagementInstance
  properties: {
    value: '<policies><inbound></inbound><backend><forward-request/></backend><outbound></outbound></policies>'
    format: 'xml'
  }
}
```

We are creating an API Management instance with a `Consumption` SKU and its corresponding policy.

## Creating an API

Add the following content to the `main.bicep` file to create one of the many APIs we can define. In this case, we are using [https://restcountries.com/](https://restcountries.com/) API:

```json
resource apiManagementApi 'Microsoft.ApiManagement/service/apis@2023-03-01-preview' = {
  parent: apiManagementInstance
  name: 'countries-api'
  properties: {
    apiRevision: '1'
    displayName: 'Contries API'
    description: 'Contries API'
    subscriptionRequired: true
    serviceUrl: 'https://restcountries.com/'
    subscriptionKeyParameterNames: {
      header: 'Ocp-Apim-Subscription-Key'
      query: 'subscription-key'
    }
    protocols: [
      'https'
    ]
    path: ''
  }
}

resource apiManagementApiPolicy 'Microsoft.ApiManagement/service/apis/policies@2020-12-01' = {
  name: 'policy'
  parent: apiManagementApi
  properties: {
    value: '<policies> <inbound> <base /> <set-header name="X-API-KEY" exists-action="append"> <value>${thirdPartyApiKey}</value> </set-header> </inbound> <backend> <base /> </backend> <outbound> <base /> </outbound> <on-error> <base /> </on-error> </policies>'
    format: 'xml'
  }
}
```

## Adding an Operation

Add the following content to the `main.bicep` file to add an operation that retrieves information about a country based on its name:

```json
resource apiManagementOperationGetCountry 'Microsoft.ApiManagement/service/apis/operations@2023-03-01-preview' = {
  parent: apiManagementApi
  name: 'get-countries'
  properties: {
    displayName: '/v3.1/name/{name} - GET'
    method: 'GET'
    urlTemplate: '/v3.1/name/{name}'
    templateParameters: [
      {
        name: 'name'
        type: 'string'
        required: true
      }
    ]
    request: {
      queryParameters: [
        {
          name: 'fullText'
          type: 'boolean'
        }
      ]
    }
    responses:[
      {
        statusCode: 200
        description: 'Success'
        representations:[
          {
            contentType:'application/json'
          }
        ]
      }
    ]
  }
}

resource apiManagementOperationGetCountryPolicy 'Microsoft.ApiManagement/service/apis/operations/policies@2020-12-01' = {
  name: 'policy'
  parent: apiManagementOperationGetCountry
  properties: {
    value: '<policies><inbound><base/></inbound><backend><base/></backend><outbound><base/></outbound><on-error><base/></on-error></policies>'
    format: 'xml'
  }
}
```

## Defining a Product

Add the following content to the `main.bicep` file to define a product and the link against our API:

```json
resource product 'Microsoft.ApiManagement/service/products@2020-12-01' = {
  name: 'product'
  parent: apiManagementInstance
  properties: {
    displayName: 'Product'
    description: 'Product'
    subscriptionRequired: true
    approvalRequired: true
    state: 'published'
    subscriptionsLimit: 1
    terms: 'These are the terms of use ...'
  }
}

resource productPolicies 'Microsoft.ApiManagement/service/products/policies@2020-12-01' = {
  name: 'policy'
  parent: product
  properties: {
    value: '<policies><inbound><base/></inbound><backend><base/></backend><outbound><base/></outbound><on-error><base/></on-error></policies>'
    format: 'xml'
  }
}

resource linkApiToProduct 'Microsoft.ApiManagement/service/products/apiLinks@2023-03-01-preview' = {
  name: 'link'
  parent: product
  properties: {
    apiId: apiManagementApi.id
  }
}
```

## Creating a Subscription

And finally, add a subscription to our product by including the following content in the `main.bicep` file:

```json
resource developerSubscription 'Microsoft.ApiManagement/service/subscriptions@2023-03-01-preview' = {
  parent: apiManagementInstance
  name: 'developer'
  properties: {
    scope: '/products/${product.id}'
    state: 'active'
    displayName: 'developer'
  }
}
```

## Deploying the script

First, create a resource group using the following command:

```powershell
az group create -l eastus -n MyResourceGroup
```

And deploy the `main.bicep` file:

```powershell
az deployment group create --resource-group MyResourceGroup --template-file .\main.bicep
```

There are several other resources to create, such as users, groups, schemas, etc. However, we believe this is a good starting point for using Azure Bicep with API Management. Thanks, and happy coding.