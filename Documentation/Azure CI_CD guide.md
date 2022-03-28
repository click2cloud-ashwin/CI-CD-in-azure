# CI/CD for Windows Containers with Azure DevOps
## Abstract
Azure DevOps enables you to host, build, plan and test your code with complimentary workflows. 
This sample application is a simple task-tracking app built with .NET Framework. The guide has a azure-pipelines.yaml (for reference) that is set up for Continuous Integration and Continuous Deployment.

## Prerequisites:
### 1. Import Repository
*   Create a project inside the Azure DevOps organization in [Azure DevOps](https://dev.azure.com). Also import the github repository of your application within the project.

### 2. Create a Service Connection
Before you create your pipeline, you should first create your Service Connection since you will be asked to choose and verify your connection when creating your template. A Service Connection will allow you to connect to your ACR when using the task templates. You can create a new Service Connection following the directions [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/service-endpoints?view=azure-devops&tabs=yaml#create-new). 

* #####  `Note: While creating service connection select Azure Resource Manager option. Also skip the resource group field in next tab.`

### 3. The Dockerfile
The samples below explain the associated Dockerfiles for the .NET Core sample applications linked above. If creating your own application, use the appropriate Dockerfile as below and replace the directory paths to match your application.
For `.NET Core`, the nano server base image must be “1809” to be compatible with what Azure currently supports. Keep in mind this may change in the future.

![image](https://user-images.githubusercontent.com/86832286/155716984-72fef396-38d9-4cce-98ac-801b99aec6cc.png)


## Create the Prerequisite Pipeline
Once you have your repository created in Azure DevOps, or imported from GitHub, you can create your pipeline. On the left menu bar go to Pipelines and click the Create Pipeline button. The next screen will ask you where the code is to create the pipeline from. We already have our code imported, so we can choose Azure Repos Git to select your current repository. As we want to setup the environment (Resource Group, ACR, AKS cluster, Azure SQL Server and Database) so we will choose the Existing Azure Pipelines YAML file that allows us to automatically create a Resource Group, ACR, AKS cluster, Azure SQL Server and Database and also create inside SQL Database.

![image](https://user-images.githubusercontent.com/69102413/160376686-41c91f37-920c-4dc5-a8f0-7e07e5c07a9d.png)

Select the branch and locate the `prerequisite-pipeline.yaml` file. Here provide the values for all the environment variables. 

![image](https://user-images.githubusercontent.com/82659622/160397341-cf527301-7248-41fa-bdbf-86e960f7b8a7.png)

Finally run the pipeline. 
* #####  `Note: This app has a data.dacpac file which is used to create Table inside Database. For other apps you need to create a dacpac file by following step.`  

### Creating a dacpac file in your project
As mentioned before, you’ll need to use either a dacpac file or set of SQL scripts to deploy your database schema. If you are using Visual Studio, it’s easy to create and add the needed dacpac file to run the action.

1. Connect your SQL Azure Database to Visual Studio
2. Right-click the data base and choose Extract Data-tier application
3. On the following window, choose the location at the same level of your github workflow file and click create.
    
  ![image](https://user-images.githubusercontent.com/82659622/157430098-c8b8d861-f56a-42c1-8302-d39d7276e9aa.png)
  
Your dacpac file should have been created and added to your project. The action finds your file under the dacpac-package parameter seen above.

## Create the Deployment Pipeline

Once the prerequisite pipeline run successfully, verify the resources are created within a resource group by signing in [Azure portal](https://portal.azure.com/#home).
Now copy the `connection string` from Azure SQL Database and paste it into database configuration file. Our sample app has a dummy connection string in the appsettings.json that will need to be changed so that the app will be connect with the database.

Now setup the azure-pipeline. On the left menu bar go to Pipelines and click the Create Pipeline button. The next screen will ask you where the code is to create the pipeline from. We already have our code imported, so we can choose Azure Repos Git to select your current repository. Since we are using Azure Kubernetes Service we can choose the Azure Kubernetes Service template that allows us to build and push an image to Azure Container Registry and deploy the application image in AKS cluster.

![image](https://user-images.githubusercontent.com/86832286/155716308-1f5f21f7-2db5-410d-8672-92d90377d36f.png)

Choose your `subscription` that you will be pushing your resources to, then pick your AKS cluster, Namespace, Container registry on the following screen. You will notice your Image Name and Service Port are pre-populated with a suggested name and port exposed in Dockerfile. You can leave those as it is, and click on the Validate and configure button to generate your azure-pipeline.yaml file. Now you can run the pipeline.

`Note:` Double check that your vmImageName = ‘windows-2019’ as it might default to ‘ubuntu-latest’.


### Summary

From here you are setup to continuously build your Windows Container application through Azure DevOps. Below you’ll see the final result of the workflow yaml file.

#### Full workflow file

The previous sections showed how to assemble the workflow step-by-step. The full `azure-pipelines.yaml` is below.

```bigquery
# Deploy to Azure Kubernetes Service
# Build and push image to Azure Container Registry; Deploy to Azure Kubernetes Service
# https://docs.microsoft.com/azure/devops/pipelines/languages/docker

# The branch that triggers the pipeline to start building
trigger:
- master

# The source used by the pipeline
resources:
- repo: self

variables:
  # Container registry service connection established during pipeline creation
  dockerRegistryServiceConnection: '<your-service-connection-number>'
  imageRepository: '<image-repository-name>'
  containerRegistry: '<your-container-registry>'
  dockerfilePath: '**/Dockerfile'
  tag: '$(Build.BuildId)'
  imagePullSecret: '<image-pull-secret>'

  # Agent VM image name
  vmImageName: 'windows-2019'

  # Name of the new namespace being created to deploy the PR changes.
  k8sNamespaceForPR: 'review-app-$(System.PullRequest.PullRequestId)'

stages:
- stage: Build
  displayName: Build stage
  jobs:
  - job: Build
    displayName: Build
    pool: 
     vmImage: $(vmImageName)      
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)

    - upload: manifests
      artifact: manifests

- stage: Deploy
  displayName: Deploy stage
  dependsOn: Build

  jobs:
  - deployment: Deploy
    condition: and(succeeded(), not(startsWith(variables['Build.SourceBranch'], 'refs/pull/')))
    displayName: Deploy
    pool: 
     vmImage: $(vmImageName)
    environment: 'Taskapp.default'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: Create imagePullSecret
            inputs:
              action: createSecret
              secretName: $(imagePullSecret)
              dockerRegistryEndpoint: $(dockerRegistryServiceConnection)

          - task: KubernetesManifest@0
            displayName: Deploy to Kubernetes cluster
            inputs:
              action: deploy
              manifests: |
                $(Pipeline.Workspace)/manifests/deployment.yml
                $(Pipeline.Workspace)/manifests/service.yml
              imagePullSecrets: |
                $(imagePullSecret)
              containers: |
                $(containerRegistry)/$(imageRepository):$(tag)


  - deployment: DeployPullRequest
    displayName: Deploy Pull request
    condition: and(succeeded(), startsWith(variables['Build.SourceBranch'], 'refs/pull/'))
    pool: 
     vmImage: $(vmImageName)

    environment: 'Taskapp.$(k8sNamespaceForPR)'
    strategy:
      runOnce:
        deploy:
          steps:
          - reviewApp: default

          - task: Kubernetes@1
            displayName: 'Create a new namespace for the pull request'
            inputs:
              command: apply
              useConfigurationFile: true
              inline: '{ "kind": "Namespace", "apiVersion": "v1", "metadata": { "name": "$(k8sNamespaceForPR)" }}'

          - task: KubernetesManifest@0
            displayName: Create imagePullSecret
            inputs:
              action: createSecret
              secretName: $(imagePullSecret)
              namespace: $(k8sNamespaceForPR)
              dockerRegistryEndpoint: $(dockerRegistryServiceConnection)

          - task: KubernetesManifest@0
            displayName: Deploy to the new namespace in the Kubernetes cluster
            inputs:
              action: deploy
              namespace: $(k8sNamespaceForPR)
              manifests: |
                $(Pipeline.Workspace)/manifests/deployment.yml
                $(Pipeline.Workspace)/manifests/service.yml
              imagePullSecrets: |
                $(imagePullSecret)
              containers: |
                $(containerRegistry)/$(imageRepository):$(tag)

          - task: Kubernetes@1
            name: get
            displayName: 'Get services in the new namespace'
            continueOnError: true
            inputs:
              command: get
              namespace: $(k8sNamespaceForPR)
              arguments: svc
              outputFormat: jsonpath='http://{.items[0].status.loadBalancer.ingress[0].ip}:{.items[0].spec.ports[0].port}'
  
              
          # Getting the IP of the deployed service and writing it to a variable for posing comment
          - script: |
              url="$(get.KubectlOutput)"
              message="Your review app has been deployed"
              if [ ! -z "$url" -a "$url" != "http://:" ]
              then
                message="${message} and is available at $url.<br><br>[Learn More](https://aka.ms/testwithreviewapps) about how to test and provide feedback for the app."
              fi
              echo "##vso[task.setvariable variable=GITHUB_COMMENT]$message"
              
```
