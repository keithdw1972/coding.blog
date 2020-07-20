> :Hero src=https://images.unsplash.com/reserve/81gZijLSWfge41LgzqQ6_Moving%20Parts.JPG?w=1900&h=600&fit=crop,
>       mode=light,
>       target=desktop,
>       leak=156px

> :Hero src=https://images.unsplash.com/reserve/81gZijLSWfge41LgzqQ6_Moving%20Parts.JPG?w=1200&h=600&fit=crop,
>       mode=light,
>       target=mobile,
>       leak=96px

> :Hero src=https://images.unsplash.com/reserve/81gZijLSWfge41LgzqQ6_Moving%20Parts.JPG?w=1900&h=600&fit=crop,
>       mode=dark,
>       target=desktop,
>       leak=156px

> :Hero src=https://images.unsplash.com/reserve/81gZijLSWfge41LgzqQ6_Moving%20Parts.JPG?w=1200&h=600&fit=crop,
>       mode=dark,
>       target=mobile,
>       leak=96px

> :Title shadow=0 0 8px black, color=white
>
> Deploy Pull Requests to Kubernetes for Review with Azure DevOps

> :Author src=github

<br>

When a Pull Request got created, you might want to try out the changes in a testing environment. Here is how to optionally deploy Azure DevOps Pull Requests to a dedicated Kubernetes namespace with a single click.

## What we need

To achieve automatic deployments of Pull Requests in Azure DevOps, we need a few things beside the obvious ones like a Kubernetes Cluster and your application as a Container with a Helm Chart or deployment definition.

- Azure Pipeline to build the app and deploy it
- An optional Build Trigger to kick off the pipeline

## Create an Azure Pipeline for Build and Deploy

We will setup Azure DevOps, to offer running an additional Build Validation Branch Policy whenever a Pull Request to a specific branch got opened. These build validations come as Azure Pipelines.

```yaml
name: $(BuildID)
trigger: none

variables:
  - name: AzureSubscription
    value: 'EVE Azure Subscription (sponsored)'
  - name: KubernetesConnection
    value: 'EVE AKS (Dev)'
  - name: ContainerRegistry
    value: 'EVE ACR DEV (new)'
  - name: KubernetesConnection
    value: ''
  - name: Tag
    value: 'pr$(System.PullRequest.PullRequestId)-$(Build.BuildId)'

stages:
  - stage: 'BuildPR'
    jobs:
      - job: Build
        pool:
          vmImage: 'Ubuntu-16.04'
        steps:
          - task: Docker@2
            displayName: 'Build and push container image'
            inputs:
              containerRegistry: $(ContainerRegistry)
              repository: 'test'
              command: 'buildAndPush'
              Dockerfile: 'src/Dockerfile'
              buildContext: '.'
              tags: |
                $(Tag)

      - job: 'DeployPR'
        displayName: 'Deploy Pull Request'
        dependsOn: [Build]
        variables:
          - name: Namespace
            value: 'myapp-pr$(System.PullRequest.PullRequestId)'
          - name: ReleaseName
            value: 'myapp-pr$(System.PullRequest.PullRequestId)'

          - task: Kubernetes@1
            displayName: 'Create Namespace'
            inputs:
              connectionType: 'Kubernetes Service Connection'
              kubernetesServiceEndpoint: $(KubernetesConnection)
              command: 'apply'
              useConfigurationFile: true
              configurationType: 'inline'
              inline: |
                apiVersion: v1
                kind: Namespace
                metadata:
                  name: '$(Namespace)'

          - task: HelmInstaller@1
            displayName: 'Install Helm'
            inputs:
              helmVersionToInstall: 'v3.1.0'

          - task: HelmDeploy@0
            displayName: 'Install Helm chart'
            inputs:
              connectionType: 'Kubernetes Service Connection'
              kubernetesServiceConnection: $(KubernetesConnection)
              namespace: $(Namespace)
              command: 'upgrade'
              chartType: 'FilePath'
              chartPath: 'env/helm/MyApp'
              releaseName: $(ReleaseName)
              overrideValues: "\
                tag=$(Tag)"
```

Once the pipeline definition is written, we can create a new Azure Pipeline from it by checking the `deploy-pr.azure-pipelines.yaml` file in to our repository and creating a new Azure Pipeline in Azure DevOps by selecting an existing YAML file.

![Create a new Pipeline in Azure DevOps from an existing YAML file](img/2020-07-20_Deploy-PR-AzureDevops_Untitled.png)

Now that the pipeline is created, we can add it to the optional steps that can be kicked off from a Pull Request check.

## Add the pipeline as a Pull Request Check

In the *Branches* section of the *Repos* submenu in Azure DevOps, we can add Policies to each branch by clicking the menu icon next to a branch and selecting the *Branch policies* option.

Here we can setup requirements for Pull Request that are being made against the selected branch. Beside policies for reviewers and status checks, we can also setup *Build Validations*, which are Azure Pipelines that run to validate the changes. Some of them can be required and run automatically, while others can be optional and only run on a user's request. The latter is exactly what we need.

Add a Build Validation policy for the Pull Request Pipeline we created earlier and declare its trigger to manual and its requirement to optional.

![Create a new optional and manual Build Validation policy for the Azure Pipeline that builds and deploys the Pull Request](2020-07-20_Deploy-PR-AzureDevops_Untitled2.png)

Now, if someone creates a new Pull Request targeting the branch you just setup the new policy for, the reviewer can chose to deploy the changes to a dedicated Kubernetes namespace for detailed review, just by kicking off the newly created pipeline.

Running the pipeline that builds and deploys the changes is a bit hidden. In the Pull Request view, click the *View all checks* button to see a list of all Build Validation pipelines that can be run for this Pull Request. Here you will find the optional one, that we just created. Once you click the Queue button, your pipeline will be executed.

![Deploy a Pull Request by queueing the optional pipeline hidden in the Checks](img/2020-07-20_Deploy-PR-AzureDevops_Untitled3.png)

## Optional: Post a link to the app to the Pull Request comments

Personally, I like not just silently deploying a Pull Request but also posing an internal link for reviewing automatically to the Pull Request's comments. I use the [Create Pull Request Comment](https://marketplace.visualstudio.com/items?itemName=CSE-DevOps.create-pr-comment-task) Task from the Azure DevOps Marketplace for that but of course you need to make sure, that your Helm Chart is built in a way, that it creates a route in your Ingress Controller for the newly deployed testing version of your app.

## Gotchas

One thing that bothers me about that solution is the fact that I have to delete the deployment manually after merging the Pull Request. Currently, there is no Trigger in Azure DevOps to run a pipeline after merging. Fingers crossed, that Azure DevOps brings a native "Deploy Pull Requests to Kubernetes" feature one day, which includes that.
