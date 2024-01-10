# azure-devops-pipelines
This repository contains various templates that can be used as a starting point for Azure DevOps projects

## Infrastructure as Code
IaC is used to manage your environment's infrastructure with the use of code.

### How to create a variable group
A variable group can be used to provide variables to your pipelines during deployments.
Secrets and environment based settings can be stored here.
As an example, you could have one per environment that stores a different connection string used by the templates:
* iac-dev
* iac-qa
* iac-prod

Go to DevOps and navigate to Pipelines > Library and click on + Variable Group
![Create variable group](/docs/img/variable-group-create-01.png)
Give the variable group a name and add a variable you want to store.
![Create variable group values](/docs/img/variable-group-create-02.png)
If this variable is sensitive information like a password, you can click on the lock icon to secure it.
NOTE: you can't retrieve the encrypted value through this screen once a value has been encrypted.
You can also set up the variable group to read from a KeyVault on Azure.

### How to create an environment
Sometimes we want to setup environments that we want to deploy to.
To create an environment, go to DevOps and navigate to Pipelines > Environments and click on Create environment.
![Create environment](/docs/img/environment-create-01.png)
Give this a name. For these demos, select None.
![Create environment](/docs/img/environment-create-02.png)
The environment can then be configured with approvals and other settings.
This can ensure that only a particular set of users can deploy to an environment.
![Environment approvals and checks](/docs/img/environment-create-03.png)

### How to create a pipeline
Go to DevOps and navigate to Pipelines > Pipelines and click on New pipeline button.
![Create pipeline](/docs/img/pipeline-create-01.png)
Select the repository that you are using for your project, in our examples we will be using GitHub.
![Select repository](/docs/img/pipeline-create-02.png)
Select your repository, we choose azure-devops-pipelines.
Then select Existing Azure Pipelines YAML file.
![Existing pipeline](/docs/img/pipeline-create-03.png)
Then select your branch and template file.
![YAML file selection](/docs/img/pipeline-create-04.png)
You can then review and save your pipeline.
![YAML file selection](/docs/img/pipeline-create-05.png)
You can also rename your pipeline after that.
![YAML file selection](/docs/img/pipeline-create-06.png)

### Using Bicep
Bicep is a declaritive language that is used to deploy resources to Azure. [What is Bicep?](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview?tabs=bicep)

We have included a sample [Bicep file](/iac/templates/bicep/main.bicep) that deploys a virtual network to Azure.
In the [stage-deploy](/iac/templates/stage-deploy.yml) template is the detailed step to deploy using Bicep.
```
- deployment: deploy_iac_job_bicep
    displayName: Deploying the infrastructure via Bicep
    environment: 'iac-${{ parameters.environment }}'
    condition: eq('${{parameters.deployType}}', 'bicep')
    variables:
      bicepTemplateFile: '$(Build.SourcesDirectory)/iac/templates/bicep/main.bicep'
      bicepParamFileJson: '$(Build.SourcesDirectory)/iac/templates/bicep/${{ parameters.environment }}-param.json'
    strategy:
      runOnce:
        deploy:
          steps:
            - checkout: self
            - task: CmdLine@2
            - task: AzureResourceManagerTemplateDeployment@3
              inputs:
                deploymentScope: 'Resource Group'
                azureResourceManagerConnection: '${{ parameters.azureServiceConnection }}'
                action: 'Create Or Update Resource Group'
                resourceGroupName: '${{ parameters.resourceGroupName }}'
                location: '${{ parameters.location }}'
                templateLocation: 'Linked artifact'
                csmFile: '${{ variables.bicepTemplateFile }}'
                csmParametersFile: '${{ variables.bicepParamFileJson }}'
                deploymentMode: 'Incremental'
                deploymentName: 'DeployPipelineTemplate'
```
Notes on the above:
* environment - this will allow us to put approval gates for specific environments in place.
* AzureResourceManagerTemplateDeployment@3 - as of 2024-01-09, this has an issue deploying .bicepparam files, hence we are using .json
* azureResourceManagerConnection - this requires a service connection set up with rights to deploy to your selected RG

### Using Azure CLI
The Azure command-line interface [(Azure CLI)](https://learn.microsoft.com/en-us/cli/azure/) is a set of commands used to create and manage Azure resources.
We can use these commands to create the resources we need in our IaC.
In the [stage-deploy](/iac/templates/stage-deploy.yml) template is the detailed step to deploy using Azure CLI.
```
- deployment: deploy_iac_job_azcli_vnet
    displayName: Deploying vnet via Azure CLI
    environment: 'iac-${{ parameters.environment }}'
    condition: eq('${{parameters.deployType}}', 'azcli')
    strategy:
      runOnce:
        deploy:
          steps:
            - task: AzureCLI@2
              displayName: Create VNET
              inputs:
                azureSubscription: '${{ parameters.azureServiceConnection }}'
                scriptType: pscore
                scriptLocation: inlineScript
                inlineScript: |
                  az network vnet create -n 'devops-examples-vn-${{ parameters.environment }}' -g '${{ parameters.resourceGroupName }}'
  - deployment: deploy_iac_job_azcli_subnet
    displayName: Deploying subnet via Azure CLI
    environment: 'iac-${{ parameters.environment }}'
    condition: eq('${{parameters.deployType}}', 'azcli')
    dependsOn: deploy_iac_job_azcli_vnet
    strategy:
      runOnce:
        deploy:
          steps:
            - task: AzureCLI@2
              displayName: Create Subnet
              inputs:
                azureSubscription: '${{ parameters.azureServiceConnection }}'
                scriptType: pscore
                scriptLocation: inlineScript
                inlineScript: |
                  az network vnet subnet create --vnet-name 'devops-examples-vn-${{ parameters.environment }}' -g '${{ parameters.resourceGroupName }}' --name 'devops-examples-sbn-${{ parameters.environment }}' --address-prefixes 10.0.0.0/24
```
Notes on the above:
* Each command can be called in it's own deployment step or together in one.
* Deploying them seperately, like above, can facilitate complex flows where some resources require a previous step to complete first, while allowing other steps to occur in parallel. Note the dependsOn section for the subnet above as an example.

### Using Azure Resource Manager templates
ARM templates are JSON files that specify the structure of resources within an Azure resource group. An overview can be found [here](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/overview).

We have included a sample [ARM template](/iac/templates/arm/main.json) that deploys a virtual network to Azure.
In the [stage-deploy](/iac/templates/stage-deploy.yml) template is the detailed step to deploy using ARM templates.
```
- deployment: deploy_iac_job_arm
    displayName: Deploying the infrastructure via ARM templates
    environment: 'iac-${{ parameters.environment }}'
    condition: eq('${{parameters.deployType}}', 'arm')
    variables:
      armTemplateFile: '$(Build.SourcesDirectory)/iac/templates/arm/main.json'
      armParamFileJson: '$(Build.SourcesDirectory)/iac/templates/arm/${{ parameters.environment }}-param.json'
    strategy:
      runOnce:
        deploy:
          steps:
            - checkout: self
            - task: CmdLine@2
            - task: AzureResourceManagerTemplateDeployment@3
              inputs:
                deploymentScope: 'Resource Group'
                azureResourceManagerConnection: '${{ parameters.azureServiceConnection }}'
                action: 'Create Or Update Resource Group'
                resourceGroupName: '${{ parameters.resourceGroupName }}'
                location: '${{ parameters.location }}'
                templateLocation: 'Linked artifact'
                csmFile: '${{ variables.armTemplateFile }}'
                csmParametersFile: '${{ variables.armParamFileJson }}'
                deploymentMode: 'Incremental'
                deploymentName: 'DeployPipelineTemplate'
```
The above is deployed in a similar fashion to the Bicep template.