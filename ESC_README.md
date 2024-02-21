# Guide to Deploy a Service in AWS ECS with Pipelines

## Prerequisites

- Make sure you have the latest versions of AWS CLI and ECS CLI installed.
- You have an AWS account and your AWS CLI is configured with the necessary credentials.

## Step 1: Create the ECR repository

First, we need to create a repository in AWS ECR to store our Docker images. You can use the following AWS CLI command:

```
aws ecr create-repository --repository-name my-repo
```

## Step 2: Create the ECS Cluster

Next, we need to create an ECS cluster. This can be done using the following AWS CLI command:

```
aws ecs create-cluster --cluster-name my-cluster
```

## Step 3: Create a Key Pair

If you need to access your ECS instances, you'll need a key pair. You can create one using the following AWS CLI command:

```
aws ec2 create-key-pair --key-name my-key-pair --query 'KeyMaterial' --output text > my-key-pair.pem
```

## Step 4: Create a CodeCommit Repository

To store your code, you need to create a CodeCommit repository. You can do this using the following AWS CLI command 

```
aws codecommit create-repository --repository-name my-repo
```

## Step 5: Update the Python Code and Dockerfile

The Python code and Dockerfile provided are fine, but we need to add a Dockerfile for the redis service:

```
# Dockerfile.redis

FROM redislabs/redismod
```

## Step 6: Build Docker Images and Push to ECR

First, check out your AWS Account ID and region:

```
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export AWS_REGION=$(aws configure get region)
```

Then, authenticate Docker to your ECR registry:

```
aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
```

Build the images:

```
docker build -t my-app -f Dockerfile .
docker build -t my-redis -f Dockerfile.redis .
```

Tag and push the images to ECR:

```
docker tag my-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-app:latest
docker tag my-redis:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-redis:latest

docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-app:latest
docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-redis:latest
```

## Step 7: Create the ECS Service

Create a task definition file `task-definition.json`:

```json
{
  "family": "my-app",
  "networkMode": "bridge",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-app:latest",
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8000
        }
      ],
      "links": [
        "redis"
      ]
    },
    {
      "name": "redis",
      "image": "$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-redis:latest",
      "essential": true
    }
  ]
}
```

Register the task definition with ECS:

```
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

Create a service on your cluster:

```
aws ecscreate-service --cluster my-cluster --service-name my-service --task-definition my-app --desired-count 1
```

## Step 8: Setup CodePipeline

In AWS Console, create a new pipeline and set the source provider to AWS CodeCommit and select the repository and branch you created earlier. 

For the build stage, choose AWS CodeBuild and create a new build project. In the build project settings, use the following buildspec:

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker images...
      - docker build -t my-app -f Dockerfile .
      - docker build -t my-redis -f Dockerfile.redis .
      - docker tag my-app:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-app:latest
      - docker tag my-redis:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-redis:latest
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker images...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-app:latest
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-redis:latest
      - echo Images pushed successfully

```

In the deploy stage, choose AWS ECS and select the cluster and service you created earlier.

Finally, review your pipeline settings and create the pipeline.

## Step 9: Test the Pipeline

Make a change to your app.py file and push the changes to your CodeCommit repository:

```
git add app.py
git commit -m "Update app.py"
git push
```

Watch the pipeline execute in the AWS Console and ensure that it completes successfully. You should see your updated application running in your ECS service.

## Conclusion

You now have a service running on AWS ECS that's automatically updated whenever you push changes to your CodeCommit repository. This pipeline helps to automate your deployment process, reducing the possibility of human error and increasing the speed at which you can deliver updates.
