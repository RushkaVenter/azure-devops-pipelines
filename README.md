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

