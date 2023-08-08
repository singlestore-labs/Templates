# <img src="https://github.com/singlestore-labs/singlestoredb-python/blob/main/resources/singlestore-logo.png" height="60" valign="middle"/> Create embeddings with a serverless architecture

## Introduction

The exploration and utilization of embeddings is a fascinating field within machine learning and data science, and is now an accessible one. Whether you are an experienced data scientist or just starting your journey in the world of embeddings, this blog post offers a comprehensive guide to creating them at scale using SingleStoreDB.

In this detailed walkthrough, we will be focusing on the integration of OpenAI's text-embedding-ada-002 model, striving to keep our solution as model-agnostic and data source-independent (still SingleStoreDB 😉) as possible.

> **Prerequisites** :
* [AWS Command Line Interface (AWS CLI) version 2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [Docker](https://docs.docker.com/get-docker/)

## Getting Started

### Build the docker image locally
Download the files or fork the repo from **src** folder on your local machine. 

Through your CLI, go to the folder where you stored theose 3 files and run the following command:
```bash
docker build -t docker-image:test .
```

### Upload the image to Amazon ECR

**To upload the image to Amazon ECR and create the Lambda function**
1. Run the get-login-password command to authenticate the Docker CLI to your Amazon ECR registry.

* Set the --region value to the AWS Region where you want to create the Amazon ECR repository.
* Replace 111122223333 with your AWS account ID.

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 111122223333.dkr.ecr.us-east-1.amazonaws.com
```

2. Create a repository in Amazon ECR using the create-repository command. Replace
```bash
aws ecr create-repository --repository-name singlestore-lambda --image-scanning-configuration scanOnPush=true --image-tag-mutability MUTABLE
```

If successful, you see a response like this:
```json
{
    "repository": {
        "repositoryArn": "arn:aws:ecr:us-east-1:111122223333:repository/singlestore-lambda",
        "registryId": "111122223333",
        "repositoryName": "singlestore-lambda",
        "repositoryUri": "111122223333.dkr.ecr.us-east-1.amazonaws.com/singlestore-lambda",
        "createdAt": "2023-03-09T10:39:01+00:00",
        "imageTagMutability": "MUTABLE",
        "imageScanningConfiguration": {
            "scanOnPush": true
        },
        "encryptionConfiguration": {
            "encryptionType": "AES256"
        }
    }
}
```

3. Copy the ```repositoryUri``` from the output in the previous step.

4. Run the docker tag command to tag your local image into your Amazon ECR repository as the latest version. In this command:

* Replace docker-image:test with the name and tag of your Docker image.
* Replace the Amazon ECR repository URI with the repositoryUri that you copied. Make sure to include :latest at the end of the URI.
  
```bash
docker tag docker-image:test 111122223333.dkr.ecr.us-east-1.amazonaws.com/singlestore-lambda:latest
```

5. Run the docker push command to deploy your local image to the Amazon ECR repository. Make sure to include :latest at the end of the repository URI.

```bash
docker push 111122223333.dkr.ecr.us-east-1.amazonaws.com/singlestore-lambda:latest
```

To update the function code, you must build the image again, upload the new image to the Amazon ECR repository, and then use the update-function-code command to deploy the image to the Lambda function.

### Create and configure the Lambda Function

Now you need to create your AWS Lambda function through the AWS console in the AWS Lambda service. 
* Select **Container image**
* Enter the name of your function. I use **singlestore_lambda**
* Select **Browse images**
* Select the repository for your image in the dropdown
* Select the image you want to. The image you just published should have the image tag latest
* Click on **Select image**
* If you have developed on ARM (like me on a Mac M1), you should select arm64 over x86_64
* Click on **Create function**

Now go to the Configuration and do the following:
* In the General configuration tab, in case you want to create a lot of embeddings at once, you can tweak the following to increase speed of ingestion:
    * Increase Timeout to 1 minute
    * Increase Memory to 500 MB
* Go to the Environment variables tab and enter the following - this is where you pass on all the environment variables from lambda_function.py:

<table>
  <thead style="background-color: grey;">
    <tr>
      <th><strong>Key</strong></th>
      <th><strong>Value</strong></th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td><strong>USERNAME</strong></td>
      <td>Your own entry (oftentimes we use admin for trials)</td>
    </tr>
    <tr>
      <td><strong>PASSWORD</strong></td>
      <td>Your username password</td>
    </tr>
    <tr>
      <td><strong>ENDPOINT</strong></td>
      <td>Your SingleStore endpoint <br>svc-XXX-dml.aws-virginia-5.svc.singlestore.com</td>
    </tr>
    <tr>
      <td><strong>CONNECTION_PORT</strong></td>
      <td>3306</td>
    </tr>
    <tr>
      <td><strong>DATABASE_NAME</strong></td>
      <td>Your own entry</td>
    </tr>
    <tr>
      <td><strong>SOURCE_TABLE_TEXT_COLUMN</strong></td>
      <td>Your own entry</td>
    </tr>
    <tr>
      <td><strong>SOURCE_TABLE_PK</strong></td>
      <td>Your own entry</td>
    </tr>
    <tr>
      <td><strong>DESTINATION_TABLE</strong></td>
      <td>Your own entry</td>
    </tr>
    <tr>
      <td><strong>LIMIT</strong></td>
      <td>1000 (but you can change it if you want to process more text at once)</td>
    </tr>
    <tr>
      <td><strong>BATCH_SIZE</strong></td>
      <td>2000 (but you can change the size depending on the speed required)</td>
    </tr>
    <tr>
      <td><strong>URL</strong></td>
      <td>https://api.openai.com/v1/embeddings</td>
    </tr>
    <tr>
      <td><strong>OPENAPI_API_KEY</strong></td>
      <td>Your OpenAI API Key</td>
    </tr>
    <tr>
      <td><strong>EMBEDDING_MODEL</strong></td>
      <td>text-embedding-ada-002</td>
    </tr>
  </tbody>
</table>

Now go to the **Test** Tab and click on **Test**. 

### Schedule your function with Amazon EventBridge

On Amazon EventBridge, go to Schedules under Scheduler and create a schedule and do the following:
* Enter a **Schedule name**
* Under **Schedule pattern**, enter the following:
    * Occurrence: Select Recurring schedule
    * Schedule type: Select Rate-based schedule
    * Under Rate expression:
        * Enter 1 as Value
        * Select minutes as Units
    * For Flexible time window, select Off
    * Select Next
* From the Templated targets, select AWS Lambda
* In Invoke, select the lambda function you have created above
    * Select Next
* Select Next (no need to change the options)

Your Lambda function will now run every 1 minute.
