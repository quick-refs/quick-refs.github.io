---
layout: draft
title: Github, AWS CodeBuild and AWS CodePipeline CI/CD
nav_order: 2
parent: Amazon Web Services (AWS)
permalink: aws/github-aws-codebuild-and-aws-codepipeline-cicd
---
# Github, AWS CodeBuild and AWS CodePipeline CI/CD

In this tutorial we will setup complete CI/CD process to deploy an Angular application from GitHub to AWS. We will go through following steps:
1. Create a simple angular application and deploy to AWS S3
2. Automate AWS S3 bucket creation to host static website using CloudFormation
3. Setup GitHub repository and generate access tokens
3. Using AWS console create AWS CodeBuild project to build, test and deploy the application
4. Write CloudFormation template to create AWS CodeBuild Project which will be triggered by GitHub pull requests
5. Once code is merged to master, Run AWS CodePipeline to deploy the application to dev, qa and prod environments.
6. BONUS: A Lambda function to post commit status update to GitHub from AWS CodePipeline


In this tutorial we will setup complete CI/CD process to deploy applications from GitHub to AWS using AWS CodeBuild and CodePipeline.

At the end, we should have a CodeBuild project triggered to run tests whenever a pull request is submitted to the GitHub repository. When the commit gets merged to master branch AWS CodePipeline should be triggered which will build, test and deploy to development, testing and production environments. Finally, we will integrate AWS CodePipeline with the GitHub Commit Status API. This way the success or failure of the CodePipeline can be shown in the GitHub Commit history to reflect successful deployment to all the environments, as well as failures. Let's get started...

### Github Repository and Personal Access Token
We will begin by creating a new github repository, for this tutorial we will name it as "github-aws-cicd".
 
Next, let's generate a personal access token, which will be required to access the GitHub repository from AWS CodeBuild and AWS CodePipeline. To generate the personal access token in Github:

1. From the drop-down in GitHub profile photo, go to Settings -> Developer Settings -> Personal access tokens.
2. Click on Generate new token and add a description, such as "AWS" or anything that helps you identify this token in future.
3. Next, in Select scopes section select "repo", which will allow full control of private repositories
4. Copy the personal access token 

### AWS System Manager Parameter Store (SSM)
AWS System Manager Parameter Store can be used to store secrets such as github access tokens. These parameters can be referenced from AWS CloudFormation, AWS CodeBuild, cli commands, scripts etc.
1. If you haven't already, login to AWS Console and search for System Manager Service. 
2. On the left side menu, click on "Parameter Store". 
3. Then, click on "create parameter", we will name this parameter "GITHUB_ACCESS_TOKEN" and it will be a SecureString type, since this is secure and needs to be encrypted.
4. Click on "Create Parameter".

### AWS CodeBuild Project 
Using AWS CodeBuild project we will run tests, build and deploy the application. We will not use any specific application for this example but simply create a Makefile with build, test and deploy targets. Which can be replaced by platform specific commands for build, test and deploys.

`./Makefile`
```bash
build:
    echo "Building application..."

test:
    echo "Running tests"
    
deploy:
    echo "Deploying application"
```

Create CodeBuild specification separately for build and deploy as following:

`./buildspec.yaml`





