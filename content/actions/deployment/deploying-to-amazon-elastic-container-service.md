---
title: Deploying to Amazon Elastic Container Service
intro: You can deploy to Amazon Elastic Container Service (ECS) as part of your continuous deployment (CD) workflows.
product: '{% data reusables.gated-features.actions %}'
redirect_from:
  - /actions/guides/deploying-to-amazon-elastic-container-service
versions:
  fpt: '*'
  ghes: '*'
  ghae: '*'
type: tutorial
topics:
  - CD
  - Containers
  - Amazon ECS
shortTitle: Deploy to Amazon ECS
---

{% data reusables.actions.enterprise-beta %}
{% data reusables.actions.enterprise-github-hosted-runners %}

## Introduction

This guide explains how to use {% data variables.product.prodname_actions %} to build a containerized application, push it to [Amazon Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/), and deploy it to [Amazon Elastic Container Service (ECS)](https://aws.amazon.com/ecs/) when there is a push to the `main` branch.

On every new push to `main` in your {% data variables.product.company_short %} repository, the {% data variables.product.prodname_actions %} workflow builds and pushes a new container image to Amazon ECR, and then deploys a new task definition to Amazon ECS.

## Prerequisites

Before creating your {% data variables.product.prodname_actions %} workflow, you will first need to complete the following setup steps for Amazon ECR and ECS:

1. Create an Amazon ECR repository to store your images.

   For example, using [the AWS CLI](https://aws.amazon.com/cli/):

   {% raw %}```bash{:copy}
   aws ecr create-repository \
       --repository-name MY_ECR_REPOSITORY \
       --region MY_AWS_REGION
   ```{% endraw %}

   Ensure that you use the same Amazon ECR repository name (represented here by `MY_ECR_REPOSITORY`) for the `ECR_REPOSITORY` variable in the workflow below.

   Ensure that you use the same AWS region value for the `AWS_REGION` (represented here by `MY_AWS_REGION`) variable in the workflow below.

2. Create an Amazon ECS task definition, cluster, and service.

   For details, follow the [Getting started wizard on the Amazon ECS console](https://us-east-2.console.aws.amazon.com/ecs/home?region=us-east-2#/firstRun), or the [Getting started guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/getting-started-fargate.html) in the Amazon ECS documentation.

   Ensure that you note the names you set for the Amazon ECS service and cluster, and use them for the `ECS_SERVICE` and `ECS_CLUSTER` variables in the workflow below.

3. Store your Amazon ECS task definition as a JSON file in your {% data variables.product.company_short %} repository.

   The format of the file should be the same as the output generated by:

   {% raw %}```bash{:copy}
   aws ecs register-task-definition --generate-cli-skeleton
   ```{% endraw %}

   Ensure that you set the `ECS_TASK_DEFINITION` variable in the workflow below as the path to the JSON file.

   Ensure that you set the `CONTAINER_NAME` variable in the workflow below as the container name in the `containerDefinitions` section of the task definition.

4. Create {% data variables.product.prodname_actions %} secrets named `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` to store the values for your Amazon IAM access key.

   For more information on creating secrets for {% data variables.product.prodname_actions %}, see "[Encrypted secrets](/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-a-repository)."

   See the documentation for each action used below for the recommended IAM policies for the IAM user, and methods for handling the access key credentials.

5. Optionally, configure a deployment environment. {% data reusables.actions.about-environments %}

## Creating the workflow

Once you've completed the prerequisites, you can proceed with creating the workflow.

The following example workflow demonstrates how to build a container image and push it to Amazon ECR. It then updates the task definition with the new image ID, and deploys the task definition to Amazon ECS.

Ensure that you provide your own values for all the variables in the `env` key of the workflow.

{% data reusables.actions.delete-env-key %}

```yaml{:copy}
{% data reusables.actions.actions-not-certified-by-github-comment %}

name: Deploy to Amazon ECS

on:
  push:
    branches:
      - main

env:
  AWS_REGION: MY_AWS_REGION                   # set this to your preferred AWS region, e.g. us-west-1
  ECR_REPOSITORY: MY_ECR_REPOSITORY           # set this to your Amazon ECR repository name
  ECS_SERVICE: MY_ECS_SERVICE                 # set this to your Amazon ECS service name
  ECS_CLUSTER: MY_ECS_CLUSTER                 # set this to your Amazon ECS cluster name
  ECS_TASK_DEFINITION: MY_ECS_TASK_DEFINITION # set this to the path to your Amazon ECS task definition
                                               # file, e.g. .aws/task-definition.json
  CONTAINER_NAME: MY_CONTAINER_NAME           # set this to the name of the container in the
                                               # containerDefinitions section of your task definition

jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest
    environment: production

    {% raw %}steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@13d241b293754004c80624b5567555c4a39ffbe3
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@aaf69d68aa3fb14c1d5a6be9ac61fe15b48453a2

      - name: Build, tag, and push image to Amazon ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build a docker container and
          # push it to ECR so that it can
          # be deployed to ECS.
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

      - name: Fill in the new image ID in the Amazon ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@97587c9d45a4930bf0e3da8dd2feb2a463cf4a3a
        with:
          task-definition: ${{ env.ECS_TASK_DEFINITION }}
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ steps.build-image.outputs.image }}

      - name: Deploy Amazon ECS task definition
        uses: aws-actions/amazon-ecs-deploy-task-definition@de0132cf8cdedb79975c6d42b77eb7ea193cf28e
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true{% endraw %}
```

## Additional resources

For the original starter workflow, see [`aws.yml`](https://github.com/actions/starter-workflows/blob/main/deployments/aws.yml) in the {% data variables.product.prodname_actions %} `starter-workflows` repository.

For more information on the services used in these examples, see the following documentation:

* "[Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)" in the Amazon AWS documentation.
* Official AWS "[Configure AWS Credentials](https://github.com/aws-actions/configure-aws-credentials)" action.
* Official AWS [Amazon ECR "Login"](https://github.com/aws-actions/amazon-ecr-login) action.
* Official AWS [Amazon ECS "Render Task Definition"](https://github.com/aws-actions/amazon-ecs-render-task-definition) action.
* Official AWS [Amazon ECS "Deploy Task Definition"](https://github.com/aws-actions/amazon-ecs-deploy-task-definition) action.