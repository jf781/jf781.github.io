---
title: Creating an environment via API's in Azure DevOps
canonical_url: 
author: Joe Fecht
date: 2020-06-12 00:00:00 -0600
categories: [Azure DevOps]
tags: [azure, Azure DevOps, API]
toc: false
---

## Goals 

We will talk about how to setup an Environment in Azure DevOps using Rest APIs.  Unfortunately there is no documented way that I've found to do this outside of the UI.  With that said, the steps described here may change at some point in the future.  If I notice this I will update this post as needed. 

If you want to cut right to the chase, the code can be found on my [GitHub](https://github.com/jf781/AzureDevopsEnvironments).  

## Azure DevOps Environments

Before we dive into how to create the Environments lets take a look at what they are.  Per [Microsoft's documentation](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/environments?view=azure-devops), an environment is a collection of resources that can be targeted by deployments jobs in a pipeline.  By leveraging an environment to deploy the resources, Azure DevOps is able to keep a history of all of the deployments, and quickly access the commits related to a deployment. 

Currently environments are only supported for Kubernetes and VM resources.  

## Creating an environment via the UI

Lets create an enviroment in the UI first so we understand the process.  We have already configured an AKS cluster in our Azure portal.  The cluster was based on this [Quickstart guide for deploy an AKS cluster](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough).  

We have signed into our Azure DevOps org and have selected our project.   Under Pipelines we select "Environments".  Then select "Create Environment".  

![UI-1-1]({{ "/assets/img/posts/2020-06-12/ui-create-env-1.png" | relative_url }})

Now lets give our environment a name, description, and select the resource type.  In our example we are going to select "Kubernetes".  

![UI-1-2]({{ "/assets/img/posts/2020-06-12/ui-create-env-2.png" | relative_url }})

Azure DevOps will now check the associated Azure Subscriptions for any resources that match what was selected.  In our example, we selected Kubernetes and it found the AKS instance "AKSCluster001".  We leave the default namespace select and validate and create.  

![UI-1-3]({{ "/assets/img/posts/2020-06-12/ui-create-env-3.png" | relative_url }})

Azure DevOps will now create our environment.  Once completed, you should see a screen similar to the one shown below.  

![UI-1-4]({{ "/assets/img/posts/2020-06-12/ui-create-env-4.png" | relative_url }})

## Components of an Environment

With the environment created there are essentially two components that we need to be aware of.  The first is the environment itself in Azure.   This is what we created in the first step, where we gave the environment a name and a description.  

The other component is the service connection.  This is how Azure DevOps authenticates with the resource.  In our example, the AKS cluster.  

![SvcCon-1-1]({{ "/assets/img/posts/2020-06-12/ui-svc-con-1.png" | relative_url }})

These two items are easy enough to create on their own.  However the part that I was unable to find any documentation on is where the service connection and the environment are associated with each other.  To figure this out we turn to our handy friend, the developer tools within our browser.  

## Determine which APIs are being called.  

Now that we know how to create an environment and the components of one, let's see how we can automate this.  To start, I created an environment again in the UI. This time I had the developer tools open in my browser so I could see which APIs were being called and the payload they were providing.  

![API-1-1]({{ "/assets/img/posts/2020-06-12/api-create-env-1.png" | relative_url }})

Our goal is to determine which URI is being called, which API version is being used and also determine what information is provided in the payload.  We will walk through this in for the process of creating the environment.  The steps are then repeated for the service endpoint as well 

Looking at the details of the of the headers for the first command we can see lots of valuable information.  We can see the API it contacting as well as the api version.   

![API-1-2]({{ "/assets/img/posts/2020-06-12/api-create-env-2.png" | relative_url }})

Moving down to the request payload, we can see it is a very simple JSON file with only the name and description of the environment.

![API-1-3]({{ "/assets/img/posts/2020-06-12/api-create-env-3.png" | relative_url }})

So now we know how to create the environment using an API.  We do the same steps for the service endpoint.  Which in this case is the next event down called "Endpoints".  And finally we do this a third time with the "Kubernetes" request.  Each time we need to ensure we are grabbing the URI, api version and the data in the payload.  

We need to pay close attention to the "Kubernetes" request since the URI requires the environment ID to be part of included. 

![API-1-4]({{ "/assets/img/posts/2020-06-12/api-create-env-4.png" | relative_url }})

## Putting this all together.  

We know the steps to create environment using the REST API, lets see if we can make a script to automate this.  

First we need to look at the payload files.  You can find the sample payload files on my [GitHub](https://github.com/jf781/AzureDevopsEnvironments/tree/master/payload-samples) that you can use as templates as well as the ones shown below.  

Below is the JSON payload to create the environment.  This one is very simple with just the name and description of the environment. 

```json
{
    "name": "K8s-Env-003",
    "description": "This automation stuff is pretty fun"
}
```

Now lets look at the service connection.  This one is much more involved but if you look at the [sample](https://github.com/jf781/AzureDevopsEnvironments/blob/master/payload-samples/aks-svc-con.json) it really is pretty easy to understand.  The Azure subscription, tenant, and Azure DevOps project ID have been removed from this example.  

```json
{
    "data": {
        "authorizationType": "AzureSubscription",
        "azureSubscriptionId": "xxx",
        "azureSubscriptionName": "JF-Lab01",
        "clusterId": "/subscriptions/xxx/resourcegroups/JF-AKS-RG/providers/Microsoft.ContainerService/managedClusters/AKSCluster001",
        "namespace": "default"
    },
    "name": "K8s-Env-003-AKSCluster001-default",
    "type": "kubernetes",
    "url": "https://akscluster001-dns-b89255db.hcp.centralus.azmk8s.io",
    "authorization": {
        "scheme": "Kubernetes",
        "parameters": {
            "azureEnvironment": "AzureCloud",
            "azureTenantId": "xxx"
        }
    },
    "serviceEndpointProjectReferences": [
        {
            "name": "K8s-Env-003-AKSCluster001-default",
            "projectReference": {
                "id": "xxx",
                "name": ""
            }
        }
    ]
}
```

Now lets look at the payload for the task that will link the service connection to the environment.  The service endpoint is not defined because we won't know it until it is created.  We have used "svcConPlaceHolder" in its place for a reason.  

```json
{
    "name":"default",
    "namespace":"default",
    "clusterName":"AKSCluster001",
    "serviceEndpointId":"svcConPlaceHolder"
}
```

Now we that we have our payload files mostly defined, lets put together a script to automate the deployment.  

One thing I want to mention before jumping into the script is there are a [few ways to authenticate with Azure DevOps](https://docs.microsoft.com/en-us/azure/devops/integrate/get-started/authentication/authentication-guidance?view=azure-devops).  We are going to use a [Personal Access Token](https://docs.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=preview-page) that is stored in an Azure KeyVault.  

Below is the script we are going to use to automate the environment creation. 

```powershell
# Define Azure DevOps and KeyVault variables 
$org = "https://dev.azure.com/JFAzDoOrg"
$project = "JFProject001"
$vaultName = "JFCoreKV"
$secretName = "AzDoPAT"

# Get and define authentication method
$azDoPAT = (Get-AzKeyVaultSecret -VaultName $vaultName -Name $secretName).SecretValueText
$BasicAuth = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(("{0}:{1}" -f '', $azDoPAT)))

$authHeader = @{
    Authorization = "Basic $BasicAuth"
}

# Create the variables used for the payload files.
$envPayload = get-content -Path ./payload-samples/env.json -Raw 
$svcConPayLoad = get-content -Path ./payload-samples/aks-svc-con.json -Raw 
$envSvcConLink = get-content -Path ./payload-samples/env-svc-con-link.json -Raw 

# Create the environment and save the output to the $env variable. 
$env = Invoke-RestMethod -Method POST `
    -Uri $org/$project/_apis/distributedtask/environments?api-version=5.0-preview.1 `
    -ContentType "application/json" `
    -Headers $authHeader `
    -body $envPayload

# Create the Service Connection and save the output to the $svcCon variable
$svcCon = Invoke-RestMethod -Method POST `
    -Uri $org/$project/_apis/serviceendpoint/endpoints?api-version=5.1-preview.2  `
    -ContentType "application/json" `
    -Headers $authHeader `
    -body $svcConPayLoad

# Define parameters for environmentId and svcConId
$envId = $env.id
$svcConId = $svcCon.id

# Update the $envSvcConLink to include the $svcConId
$envSvcConLinkPayload = $envSvcConLink.replace("svcConPlaceHolder", $svcConId)

# Create the link between the service connection and the environment.  Note the $envId variable in the URI
Invoke-RestMethod -Method POST `
    -Uri $org/$project/_apis/distributedtask/environments/$envId/providers/kubernetes?api-version=5.0-preview.1  `
    -ContentType "application/json" `
    -Headers $authHeader `
    -body $envSvcConLinkPayload
```

It gets the personal access token from the defined Key Vault and then converts it to a base64 value to use for authentication.  It then gets the data from our payload files we listed above.  Finally it create a REST api call to the URI's we identified above with the associated payload.   

We save the output of the environment and service connections API calls as we will need them for the third API to create the link between the service connection and the environment.  We use the environment ID in the URI to create the link.  We use the "replace" method to update the payload data to replace "svcConPlaceHolder" with the actual service connection ID. 

When we run this we should get output similar to below. 

```bash
tags                 : {}
namespace            : default
clusterName          : AKSCluster001
serviceEndpointId    : f89fcbba-8bfb-4a63-
id                   : 4
name                 : default
type                 : kubernetes
createdBy            : blahblah
createdOn            : 6/12/2020 4:48:53 PM
lastModifiedBy       : blahblah
lastModifiedOn       : 6/12/2020 4:48:53 PM
environmentReference : @{id=4; name=K8s-Env-003}
```

And if we check the environments in Azure DevOps we should we our new environment

![API-1-5]({{ "/assets/img/posts/2020-06-12/api-create-env-5.png" | relative_url }})

Hopefully this helps understand how to create an environment via the API.  This method can be used for most other things as in Azure as well.

If you have any questions don't hesitate to reach out.  Keep learning!!