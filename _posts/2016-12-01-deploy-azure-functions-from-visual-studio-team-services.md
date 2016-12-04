---
layout: post
title: "Deploying Azure Functions with VSTS"
description: "Deploying Azure Functions from Visual Studio Team Services."
category: "Azure"
tags: [Azure, Azure Functions, VSTS]
---

{% include JB/setup %}

# Background

We've been looking forward to using Azure Functions in Production since we first heard about them. Now they've hit GA it’s the perfect time to have a look at integrating them into our Development and Release pipelines.

At the time of writing this article the Visual Studio 2015 tooling for Azure Functions was not available.

*Actually they released the preview today and while it looks like a great start there are logistics issues with the lack of tooling - I'm positive this will be rectified in the coming weeks and months!*

All our infrastructure is defined by ARM templates and VSTS handles our build and deployments so it's essential Functions fit into this pipeline. No manual setup. No messing around with the Azure Portal to configure deployments. 

This is a big part of our culture at work. We want very solid build and release pipelines that fit a large scale enterprise solutions like ours.

While we love the simplicity of the click-to-configure that the Azure Portal gives us for small projects, we feel these types of actions should be provisioned automatically via the appropriate tools so we can automatically replicate,  destroy and recreate without worrying about manual action (and therefore human error).

# Functions Under The Hood

Something that's quite evident when working with Functions is they are effectively deploy as Web Apps. They consists of :

• A Service Plan - this can be a Dynamic Service plan set to the Consumption Tier (pay for what you execute).

• A Function App - in terms of infrastructure this draws many similarities with a standard Web App runs on a Service Plan.

The fact that a Function App shares the same idiosyncrasies Web Apps is great! The Function App runs the same stuff underneath with a few tweaks which means it has the same file system mechanics, the same deployment strategies work, the ARM template definitions are similar, the list goes on…

Here's a great example of how similar Azure Functions are by viewing the underling Function App settings in the Portal:

Find the Mange Tab in the Function App Settings:

![alt text](/images/20161201/1.png "Azure Function App Settings"){: .col-lg-12 }


Notice anything familiar?


![alt text](/images/20161201/2.png){: .col-lg-12 }


In Visual Studio however, the Web App similarities end abruptly.

There many areas of questions when I was adding Azure Functions to our Build and Release pipeline but a couple of highlights relevant to this post:

• Where do you put your Functions in Visual Studio? 

• How do you use your own projects shared assemblies with them?

• Do I have to script things to move them about in deployment scenarios?

For me, Functions have to be in our Visual Studio Solution. This keeps our code organized, and reduces the potential for messy repository semantics.

Using CSX requires a direct reference to the external DLLs (Solution based or from NuGet). 

 At the time of writing there's no Azure Functions Visual Studio Project, and certainly no dedicated deployment mechanic in VSTS.

## ARM Template and Environment Variables

Important: Currently you have to deploy Dynamic App Service plans into Resource Groups first if they contain any standard tier Service Plans for other Web Apps. If you're adding Functions to an already existing Resource Group in Production you will have to deploy a new Resource Group. If you attempt to do this you may find the Azure Deployments fail with strange errors such as the "A Web App with the same name already exists".

In the resources section of an ARM Template we'll need a Service Plan, the Application, and if ideally a dedicated Storage Account for storing the Function Apps diagnostic and Metric Information. We use other Queue Bindings from different storage accounts to trigger the functions.

**The Service Plan (Server Farm)**

As mentioned earlier, the Service Plans for functions need to be deployed first to a resource group as Azure allocates these slightly differently from legacy systems. The key here is the SKU which is set to Dynamic with a couple of other Consumption Plans specific properties. Apart from the obvious other properties are the same as a standard service plan.

```json
{
    "type": "Microsoft.Web/serverfarms",
    "apiVersion": "2015-08-01",
    "name": "[variables('serverfarms_FunctionsPlan_name')]",
    "location": "[resourceGroup().location]",
    "sku": {
      "name": "Y1",
      "tier": "Dynamic",
      "size": "Y1",
      "family": "Y",
      "capacity": 0
    },
    "properties": {
      "name": "[variables('serverfarms_FunctionsPlan_name')]"
    }
  },
```

**Dedicate Storage Account**

This is a requirement, having dedicated storage accounts for each Function App is a good idea. I wouldn't store all your Function Apps diagnostic logs in a single storage account for various reasons. In any case, you're only billed for what you use with storage accounts also.

```json
/* Function App Dedicated Storage Account */
{
  "type": "Microsoft.Storage/storageAccounts",
  "name": "[variables('function_storage_name')]",
  "apiVersion": "2015-06-15",
  "location": "[resourceGroup().location]",
  "properties": {
    "accountType": "Standard_GRS" /* GRS */
  }
},
```

**Function App**

Again, the similarities between Web Apps and Function Apps are apparent when comparing the difference in ARM Templates. The Function App really is just a change in the "Kind" property as you can see.

The fact functions have the standard App Settings resource available makes them easy to work when configuring Environment Variables, Connection Strings and similar configuration sections Azure Veterans are used to when deploying via ARM.

```json
/* Functions App */
{
  "apiVersion": "2015-08-01",
  "type": "Microsoft.Web/sites",
  "name": "[variables('function_app_name')]",
  "location": "[resourceGroup().location]",
  "kind": "functionapp",
  "properties": {
    "name": "[variables('function_app_name')]",
    "serverFarmId": "[resourceId('Microsoft.Web/serverfarms', variables('serverfarms_FunctionsPlan_name'))]"
  },
  "dependsOn": [
    "[resourceId('Microsoft.Web/serverfarms', variables('serverfarms_FunctionsPlan_name'))]",
    "[resourceId('Microsoft.Storage/storageAccounts', variables('function_storage_name'))]"
  ],
  "resources": [
    {
      "apiVersion": "2016-03-01",
      "name": "appsettings",
      "type": "config",
      "dependsOn": [
        "[resourceId('Microsoft.Web/sites', variables('function_app_name'))]",
        "[resourceId('Microsoft.Storage/storageAccounts', variables('function_storage_name'))]"
      ],
      "properties": {
        "AzureWebJobsStorage": "[concat('DefaultEndpointsProtocol=https;AccountName=',variables('function_storage_name'),';AccountKey=',listkeys(resourceId('Microsoft.Storage/storageAccounts', variables('function_storage_name')), '2015-05-01-preview').key1,';')]",
        "AzureWebJobsDashboard": "[concat('DefaultEndpointsProtocol=https;AccountName=',variables('function_storage_name'),';AccountKey=',listkeys(resourceId('Microsoft.Storage/storageAccounts', variables('function_storage_name')), '2015-05-01-preview').key1,';')]",
        "FUNCTIONS_EXTENSION_VERSION": "~1",

        "APPINSIGHTS_INSTRUMENTATIONKEY": "[parameters('APPINSIGHTS_INSTRUMENTATIONKEY')]",

        "YOUR_TEMPLATE_PARAMETER": "[parameters('YOUR_TEMPLATE_PARAMETER')]",
      }
    }
  ]
}
```

## VSTS Deployment Solution

• Create an ASP.NET Core Project.

• Add your Functions as folders.

• Alter the project.json to include our Function folders.

• Build using Dotnet Publish and Archive the output directory.

• Deploy in VSTS using the standard Web App Deploy step.

This also allows us to Add References to our Solution Assemblies and NuGet Packages.

### ASP.NET Core Project

Create a Blank ASP.NET Core Project. I removed the wwwroot directory also. Add an Azure Function Folder with an appropriate name. Here we've got a timer based Azure Function that I copied from the Azure Functions Quick Start portal.

![alt text](/images/20161201/3.png)


The folder contains the CSX file (run.csx) and the function.json which contains bindings and timer configuration.

![alt text](/images/20161201/4.png)
![alt text](/images/20161201/5.png)


### Add your Function Folders to the Publish Options Include

This is important because we will use the `dotnet publish` command to output the publish folder we'll want to Zip up and deploy to the Function App. If you were to look in the publish output folder you'll find you're pleasantly (un)surprised to see an almost flat directory structure - this is great because we can comfortably reference our DLLs without worrying about folder structure. It should `just work ™`.


![alt text](/images/20161201/6.png)

### Build in VSTS

Seeing as our Azure Functions project (a pretend ASP.NET Core project) is contained within the solution, a solution build will adequately restore and reference our external assemblies. Combine this with a dotnet publish and the output directory is a winner for deploying to our Function App.


![alt text](/images/20161201/7.png)
![alt text](/images/20161201/8.png)


### Archive the Output

The output path of the publish will be different depending on the version of .NET you are referencing in the ASP.NET Core project. We're using .NET 4.6.2 here so the path of the directory reflects this:


![alt text](/images/20161201/9.png)

### Releasing

Assuming you have your Infrastructure deployed then deploying becomes a straight forward Web App Deployment:

![alt text](/images/20161201/10.png)


Choose the name of the deployed App Service name (whatever you chose in your ARM template) and point it to the Zip package contained within the artefacts:

![alt text](/images/20161201/11.png)


All being well you should see your Function appear in the Portal!

![alt text](/images/20161201/12.png)

## Conclusion

Functions are still a new concept to a lot of engineers and ops engineers. Microsoft have done a great job using the foundations they've built into Azure already to deploy new services that fit into existing infrastructure without re-inventing their own wheel so to speak. The tooling for VSTS and Visual Studio will definitely get better in the coming weeks and months but until then this solution works great for deploying Functions to Production through ARM Templates and VSTS without manual intervention and integrating into an existing fully automated pipeline.

As soon as Microsoft's Tooling comes out of preview, we'll switch but for now they've made it dead simple to work around the lack of tooling (right now) and get these bad boys to production!

If anyone has any questions or comments, let me know via twitter or comments!
