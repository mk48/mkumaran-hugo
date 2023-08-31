---
title: Deploye nextjs app in Azure devops
date: '2023-05-09'
tags: ["NextJs", "Devops"]
draft: false
---

## Introduction

Deploying a Next.js application to production requires a streamlined process that ensures the application is built correctly and deployed to a reliable hosting environment. Azure DevOps provides a powerful set of tools to automate these processes, ensuring efficient deployment of your Next.js app. This guide walks you through the steps to deploy your Next.js app using Azure DevOps.

## Configuring the Next.js Project

To prepare your Next.js app for deployment, follow these steps:

### 1. Update next.config.js

In your `next.config.js`, include the `output` configuration as `standalone` to ensure your app is built as a standalone bundle:

```js
const nextConfig = {
  reactStrictMode: true,
  output: 'standalone', // Add this line
  // ... other configurations
};

module.exports = nextConfig;
```
### 1. Build the Next.js App
Run the command `npm run build` to build your Next.js app. After the build, you'll find a new folder named `standalone` within the `.next` directory.
![Standalone folder](standalone-folder.png)


### 3. Copy Required Folders

Copy the following folders into the `standalone` folder:

1. Copy the `public` folder into `./next/standalone`.
2. Copy the `.next/static` folder into `./next/standalone/static`.

![Copy folders into standalone](copy-folders-into-standalone.png)
This step ensures that all the necessary assets are correctly placed for deployment.

## Running the Standalone Build

To confirm that your standalone build works as expected, follow these steps:

1. Open a command prompt within the `standalone` folder.
2. Run the command `node server.js` to start the server.
3. Open a web browser and navigate to `localhost:3000` to ensure that the website functions properly.

Remember that only the content of the `standalone` folder will be deployed.

## Setting up the Azure DevOps Pipeline
Follow these steps to create a deployment pipeline in Azure DevOps:

### 1. Create a New Pipeline


1. Go to Azure DevOps and navigate to Pipelines, then click on "New Pipeline".
![New pipe line](new-pipe-line-button.png)

2. Select "Azure Repos Git" as the source.
![Select Azure repos git](select-azure-repos-git.png)

3. Choose your repository.

### 2. Select Node.js and Configure
In the configuration section, select "Node.js" as the template.
![Select nodejs pipe line](select-nodejs-pipeline.png)

 Review the generated YAML code and click "Save and run". You can edit the configuration later if needed.
![Review yaml pipe line code](review-yaml-pipeline-code.png)
You can edit the config file now or later. If you do later then you have to check-in the changes.

Here's an example YAML configuration:

```yml
trigger:
  - none

pool:
  vmImage: ubuntu-latest

steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '18.x'
    displayName: 'Install Node.js'

  - script: |
      npm install
    displayName: 'npm install'

  - task: Cache@2
    displayName: 'Cache .next/cache'
    inputs:
      key: 'next | $(Agent.OS) | package-lock.json'
      path: '$(System.DefaultWorkingDirectory)/.next/cache'

  - script: |
      npm run build
    displayName: 'npm build'

  - task: CopyFiles@2
    displayName: 'Copy standalone'
    inputs:
      sourceFolder: '$(Build.SourcesDirectory)/.next/standalone'
      Contents: '**'
      TargetFolder: '$(Build.ArtifactStagingDirectory)/standalone'

  - task: CopyFiles@2
    displayName: 'Copy public'
    inputs:
      sourceFolder: '$(Build.SourcesDirectory)/public'
      Contents: '**'
      TargetFolder: '$(Build.ArtifactStagingDirectory)/standalone/public'

  - task: CopyFiles@2
    displayName: 'Copy .next/static'
    inputs:
      SourceFolder: '$(Build.SourcesDirectory)/.next/static/'
      Contents: '**'
      TargetFolder: '$(Build.ArtifactStagingDirectory)/standalone/.next/static'

  - task: ArchiveFiles@2
    inputs:
      rootFolderOrFile: '$(Build.ArtifactStagingDirectory)/standalone'
      includeRootFolder: false
      archiveType: 'zip'
      archiveFile: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'
      replaceExistingArchive: true

  - task: PublishPipelineArtifact@1
    displayName: 'Publish artifact'
    inputs:
      targetPath: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'
      artifact: '********'        <-- give your own name
      publishLocation: 'pipeline'
```

give a commit message and click 'save and run'
![Give a commit message](Give-commit-message.png)

it's running, once done you can see success tick and published artifact.
![Pipeline running completed](pipeline-completed.png)

### 3. Create the Azure App Service

In azure, we have to create a new `App service`
![Create new app service basic tab](new-app-service-starting-config.png)

In the next deployment tab, select disable, because we are not using GitHub's action.
![Don't select github action](dont-select-github-ci-cd.png)

Below is the final review+create screen for full information.
![Final step to create app service](final-step-to-create-app-service.png)

Once the service is created, you can see the running status.
![App service created](app-service-created.png)

### Create slots for staging

I already created app service with free plan which doesn't support slots, so I changed to paid plan (Standard S1). So Before continue slots, you have to change your app service plan to paid one.

Open the created app service, go to deployment slots section, add slot.
![Create slots for staging](slots-for-staging.png)

1. Give a name for the slot.
2. Select the main app service in the clone settings dropdown.
3. Click add.
![Name for slot](name-for-slot.png)

Once that's done, we can see one more app service with `slot`
![Deployment slots](deployment-slots.png)

Give startup command in both stagin and production slots.
![Startup command](startup-command.png)

Below settings to view the container logs
![Settings to view container logs](settings-to-view-container-log.png)

## Create new Release

### Create staging release pipeline

Click Release, and new then `New release pipeline`
![Release new pipeline](release-new.png)

Select NodeJs as a template.
![NodeJs from pipeline](select-nodejs-from-pipeline.png)

Give a name for stage, like test, dev, or pre-prod whatever. Then close this popup.
![Name for staging env](give-name-for-staging.png)

Name for the release pipeline
![Name for the release pipe line](name-for-the-release-pipeline.png)

1. Select Artifact
2. Select build pipeline
3. Give a source alias

![Select artifact](select-artifacts.png)

1. Select staging
2. Go to tasks tab

![Move to tasks tab](move-to-tasks-tab.png)

In parameter section click `unlink all`. I don't have that in the below screen.
![Unlink parameter](unline-in-parameter.png)

1. Select run on agent
2. Give a name for display
3. Select azure pipelines
4. Select `ubuntu latest`
5. Ensure your artifact selected

![Staging release settings](staging-release-settings.png)

Make changes in the `Deploy azure app service` like below image
![App service deployment setttings](app-service-deploy-settings.png)

### Create production release pipeline

Clone staging to create production release
![Clone staging pipeline](clone-staging-for-production.png)

Change the stage name as production
![Change stage name](changing-stage-name.png)

It's important to change the production's slot as production.
![Change slot as production](change-production-slot.png)

Create relase
![Create release](create-release.png)

Select all, we will do manual trigger.
![Manual trigger](manual-trigger.png)

Once created the release, click on name to open
![Release created](release-created.png)

I have already deployed, so it is showing as redeploy, but first it should be `deploy` just click to deploy.
![Depoly button](deploy-button.png)

Once deployment done, we can see the log in `Log stream`
![Listening on port 8080](listening-port-8080.png)


## Conclusion

By following these steps, you can successfully deploy your Next.js app using Azure DevOps. This automated deployment process ensures consistency and reliability in your deployments, allowing you to focus on building and improving your application.