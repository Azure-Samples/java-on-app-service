# CI/CD with Jenkins

Deploying to App Service from Jenkins is made easy with the App Service plugin. If you already have a Jenkins server, follow [these instructions](https://docs.microsoft.com/en-us/azure/jenkins/deploy-jenkins-app-service-plugin) to get started with the plugin.

If you do not already have a Jenkins server, you can [create one from the Azure Marketplace](https://docs.microsoft.com/en-us/azure/jenkins/install-jenkins-solution-template). Once your Jenkins server is up and running, follow the previous link to configure the App Service plugin.

## Overview

To use the Jenkins plugin for App Service, you will need to create a service principal (also known as an App Registration) with Contributor rights. The Jenkins plugin will use this principal to authenticate with Azure and deploy your Java app to App Service.