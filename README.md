# Azure Data Factory self-hosted integration runtime in Azure Container Instances(ACI) service

This sample illustrates how to host an [Azure Data Factory self-hosted integration runtime](https://docs.microsoft.com/azure/data-factory/concepts-integration-runtime) in Azure Container Instances service.

By using this approach, you can gain the benefits of 
* using a self-hosted integration runtime while avoiding having to manage virtual machines or other infrastructure
* orchestrating ACI up and down when needed

## Related work

* This sample is fork from [https://github.com/Azure-Samples/azure-data-factory-runtime-app-service](https://github.com/Azure-Samples/azure-data-factory-runtime-app-service)

## Approach and architecture

This sample runs the self-hosted integration in a Windows container on ACI. Azure Data Factory [supports running a self-hosted integration runtime on Windows containers](https://docs.microsoft.com/azure/data-factory/how-to-run-self-hosted-integration-runtime-in-windows-container), and [they provide a GitHub repository](https://github.com/Azure/Azure-Data-Factory-Integration-Runtime-in-Windows-Container) with a Dockerfile and associated scripts. Azure Container Registry builds the Dockerfile by using [ACR tasks](https://docs.microsoft.com/azure/container-registry/container-registry-tasks-overview).

The ACI uses [VNet integration](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-vnet) to connect to a virtual network. This means that the self-hosted integration runtime can [connect to Data Factory by using a private endpoint](https://docs.microsoft.com/azure/data-factory/data-factory-private-link), and it can also access servers and other resources that are accessible thorugh the virtual network.

To illustrate the end-to-end flow, the sample deploys an example Data Factory pipelines which
* starts ACI and polls integration runtime(IR) status until IR is available 
* connects to a web server on a virtual machine by using a private IP address.

### Architecture diagram

![Architecture diagram](architecture-diagram.png)

These are the data flows used by the solution:

1. When the ACI starts, ACI pulls the container image from the container registry.
    - The ACI ACR's admin credentials to pull the image. This has been selected to simplify the demo.
1. After the container is started, the self-hosted integration runtime loads. It connects to the data factory by using a private endpoint.
1. When the data factory's pipeline runs, the self-hosted integration runtime accesses the web server on the virtual machine.

## Deploy and test the sample

The entire deployment is defined as a Bicep file, with a series of modules to deploy each part of the solution.

To run the deployment, first create a resource group, such as by using the following Azure CLI command:

```azurecli
az group create \
  --name SHIR \
  --location australiaeast
```

Next, initiate the deployment of the Bicep file. The only mandatory parameter is `vmAdminPassword`, which must be set to a value that conforms to the password naming rules for virtual machines. The following Azure CLI command initiates the deployment:

```azurecli
az deployment group create \
  --resource-group SHIR \
  --template-file deploy/main.bicep \
  --parameters 'vmAdminPassword=<YOUR-VM-ADMIN-PASSWORD>' ['irNodeExpirationTime=<TIME-IN-SECONDS>'] ['triggerBuildTask=<true/false>']
```

where the optional parameter `irNodeExpirationTime` specifies the time in seconds when the offline nodes expire after App Service stops or restarts. The expired nodes will be removed automatically during next restarting. The minimum expiration time, as well as the default value, is 600 seconds (10 minutes).

The deployment takes approximately 30-45 minutes to complete. The majority of this time is the step to build the container image within Azure Container Registry. To decrease deployment time after first deployment `triggerBuildTask` parameter can be changed to false to disable automated container build on the ACI.

After the deployment completes, wait about another 10-15 minutes for App Service to deploy the container image. You can monitor the progress of this step by using the Deployment Center page on the App Service app resource in the Azure portal.

To test the deployment when it's completed:

1. Open the Azure portal, navigate to the resource group (named *SHIR* by default), and open the data factory.
1. Select **Open Azure Data Factory Studio**. A separate page opens up in your browser.
1. On the left navigation bar, select **Author**.
1. Invoke Azure Data Factory pipelines
  1. Under **Pipelines**, select **Start ACI**. On the toolbar, select **Debug** to start the pipeline running.
    1. The task shows the *Succeeded* status. This means the self-hosted integration runtime is running and successfully connected with Azure Data Factory service. SHIR is available for other pipelines.
  1. Under **Pipelines**, select **sample-pipeline**. On the toolbar, select **Debug** to start the pipeline running.
    1. The task shows the *Succeeded* status. This means the self-hosted integration runtime successfully connected to the web server on the virtual machine and accessed its data.

## Security

To simplify templating ACI admin username and password are used:
* Linux container on ACI [support](https://learn.microsoft.com/en-us/azure/container-instances/using-azure-container-registry-mi)  ACI with MSI. Windows containers do not support MSI at all
* The recommended alternative login credentials for ACI is service principal:
  * [ACI documentation](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-using-azure-container-registry)
  * [ACR documentation](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal)
  * Conflicting [ACR documentation](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-auth-aci) mentions usage of service principals only for ACI


## Note

In this sample, we use ACI to host the container because the other serverless/PaaS container hosting options in Azure don't support or support has limitations VNet integration with Windows containers (at least at the time of writing, June 2023).

App service is not really serverless: it is generating costs 24/7 as soon as `Microsoft.Web/serverfarms` is being deployed. Official app service [demo](https://github.com/Azure-Samples/azure-data-factory-runtime-app-service)

TODO: comments about AKS. New free tier allows running small Kubbernetes cluster with 24/7 cluster management cost. [(Released 02/2023)](https://techcommunity.microsoft.com/t5/apps-on-azure-blog/azure-kubernetes-service-free-tier-and-standard-tier/ba-p/3731432)

## TODO

* Test if ACI can pull images through a VNet-integrated container registry
* Refactor templating when Bicep support resources as module inputs and outputs
* MSI support as soon as Windows ACI supports MSI
