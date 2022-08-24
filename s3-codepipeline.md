
# Automate static website deployments to S3 using CodePipeline

This lab walks you through the steps to host and deploy [static websites to S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html) using CodePipeline. A static website comprises of individual webpages including static content. They might also contain client-side scripts. [AWS CodePipeline](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html) is a continuous delivery service you can use to model, visualize, and automate the steps required to release your software. 

This lab is set to Level 100, hence participants are not required to have any specific knowledge of AWS. The lab will take ~15 minutes to complete.


## Let's get started
You will first setup a bucket to host static website. Then you will create a deployment pipeline to deploy files, such as static website content or artifacts from your build process, to Amazon S3.

Before we begin, we will need to login into AWS Cloud9. Cloud9 is a cloud-based integrated development environment (IDE) that lets you write and run code with just a browser. All code required to setup the lab is prepared, participants are only required to run the scripts in Cloud9.

To access Cloud9 search for `Cloud9` in the AWS Console and click on Cloud9.

![Cloud9 Search](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/cloud9_search.png)

Under `Your environments`, open the **Website deployment using Codepipeline (Level 100)** Cloud9 instance by clicking on **Open IDE**

![Open IDE](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/cloud9_open_ide.png)


### Step 1: Refresh your environment
Please refresh your environment below by running the following commands:
```bash
aws cloudformation delete-stack --stack-name 'devlab-s3-bucket'
aws cloudformation delete-stack --stack-name 'devlab-s3-bucket-pipeline'
rm -rf ~/environment/devlab-s3-bucket-pipeline-website
cd ~/environment/devlab-s3-codepipeline
```

Go to [CloudFormation console](https://console.aws.amazon.com/cloudformation) and check that the stacks `devlab-s3-bucket` and `devlab-s3-bucket-pipeline` does not exist. If either stacks are there and in process of getting deleted, wait for the deletion to complete before proceeding to the next steps.

![CFN Empty Stack](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_empty_stack.png)

### Step 2: Setup S3 bucket to host your website 

To host a static website, you configure an Amazon S3 bucket for website hosting, and then upload your website content to the bucket. This bucket must have public read access. It is intentional that everyone in the world will have **read** access to this bucket. The website is then available at the AWS Region-specific website endpoint of the bucket.

1. Copy, paste and run the following command in Cloud9 terminal to create S3 bucket and set appropriate permission to host a website.

```bash
aws cloudformation deploy --template-file ~/environment/devlab-s3-codepipeline/templates/setup_s3_bucket.yaml --stack-name devlab-s3-bucket --capabilities CAPABILITY_IAM
```

![CFN Stack Deploy](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_deploy_stack.png)



AWS CloudFormation creates the AWS resources as defined in the template, and groups them in an entity called a stack in AWS CloudFormation. You can access this stack in the [CloudFormation console](https://console.aws.amazon.com/cloudformation). Click on the stack name **devlab-s3-bucket** (or filter by name if your stack is not listed). 


![CFN Stack](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_cloudformation_stack.png)

Select the **Outputs** tab. 

![CFN Stack Output](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_cloudformation_stack_output.png)



2. Copy the `WebsiteS3Bucket` value in the outputs section, which is your bucket name. Then switch to the [S3 console](https://console.aws.amazon.com/s3) and search for your bucket with the S3 console.

![S3 Search Bucket](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_search_bucket.png)


3. Click on bucket name and review Bucket policy under permissions tab. **Note** Only public read (GET) is allowed.


![S3 Review Policy](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_bucket_policy.png)


### Step 3: Upload files to S3 and access them using Website URL

1. Now you have the bucket, let us upload some files and view them in a browser. Replace the **&lt;WebsiteS3Bucket&gt;** with the `WebsiteS3Bucket` value you copied from the CloudFormation output in the commands below. Now run following commands to upload files from template folder to S3 bucket you just created. 


```bash
aws s3 cp ~/environment/devlab-s3-codepipeline/templates/index.html s3://<WebsiteS3Bucket>

aws s3 cp ~/environment/devlab-s3-codepipeline/templates/error.html s3://<WebsiteS3Bucket>
```
**Note** If you do not have the bucket name or website url, execute following command to list them. 

```bash
aws cloudformation describe-stacks --stack-name 'devlab-s3-bucket' --query Stacks[*].Outputs[*]
```

![S3 Lab Describe Stack](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_describe_stack.png)

2. Finally access the website URL in a browser and you will see a hello world message (from index.html). Your website url is the value of the `WebsiteHttpUrl` outputted by the `devlab-s3-bucket` stack.

![S3 Lab Hello World](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_hello_world.png)



### Step 4: Add code repository and pipeline to automate deployments

1. Website works fine, however everytime every update requires you to manually upload the files to S3. Lets automate this. In the following steps, we will create a code repository and a pipeline to automate deployments to S3. 


2. Execute following command to create a code commit repository and pipeline. Remember to replace the **&lt;WebsiteS3Bucket&gt;** text below with the `WebsiteS3Bucket` value you copied earlier from the CloudFormation output.

```bash
aws cloudformation deploy --template-file ~/environment/devlab-s3-codepipeline/templates/setup_deployment_pipeline.yaml --stack-name devlab-s3-bucket-pipeline --parameter-overrides WebsiteS3Bucket=<WebsiteS3Bucket> --capabilities CAPABILITY_IAM
```

**Note** `WebsiteS3Bucket` is the same bucket we created earlier. If you do not have the bucket name or website url, execute following command to list them.

```bash
aws cloudformation describe-stacks --stack-name 'devlab-s3-bucket' --query Stacks[*].Outputs[*]
```

You can review the stack in the [CloudFormation console](https://console.aws.amazon.com/cloudformation) or execute following command to review the outputs

```bash
aws cloudformation describe-stacks --stack-name 'devlab-s3-bucket-pipeline' --query Stacks[*].Outputs[*]
```

![S3 Lab Deployment Stack](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_deployment_pipeline_stack.png)


3. Now the code repository is ready, you need to [setup credentials](https://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-https-unixes.html#setting-up-https-unixes-credential-helper) so we can use access it.

```bash
git config --global credential.helper '!aws codecommit credential-helper $@'
git config --global credential.UseHttpPath true
```

4. Clone the respository using http url. This will create a new folder in our workspace.

**Note** Execute following command to fetch code commit details 

```bash
aws cloudformation describe-stacks --stack-name 'devlab-s3-bucket-pipeline' --query Stacks[*].Outputs[*]
```

 Replace the text `<CodeCloneHttpUrl>`  in the commands below with the output value of `CodeCloneHttpUrl` from the CloudFormation stack and then run the following commands:
```bash
cd ~/environment
git clone <CodeCloneHttpUrl>
```

![S3 Lab Clone Repo](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_cloned_repo.png)


5. Use following commands below to move the html files from templates folder to our repository

```bash
cp -R ~/environment/devlab-s3-codepipeline/templates/*.html ~/environment/devlab-s3-bucket-pipeline-website/
```

6. Expand the `devlab-s3-bucket-pipeline-website` folder to verify the files are there, and make changes to `index.html` using Cloud9. You can change the text in the file from `Hello World` to `Hello from Sydney Summit`. The use following commands to commit your changes to the repository. 
   

```bash
cd ~/environment/devlab-s3-bucket-pipeline-website/ 
git add .
git commit -m 'first commit: updates to index.html'
git push
```

7. Access [CodePipeline](https://console.aws.amazon.com/codepipeline) through  theconsole. You will notice that pipeline has pushed your changes to s3 bucket.

![S3 Lab Pipeline](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_pipeline.png)

8. Visit the website url again (or refresh the page if you have it opened). You should see the updated message. Your website url is the value of the `WebsiteHttpUrl` outputted by the `devlab-s3-bucket` CloudFormation stack.

![S3 Lab Updated website](https://github.com/calvngo/devlab-s3-codepipeline/blob/main/images/s3_lab_updated_website.png)


**Congratulations** You have successfully automated deployments to your S3 bucket. 


### Step 5: Review and Cleanup 

1. Congragulations you have created and configured S3 bucket to host static websites, and configured a pipeline to auto deploy changes to S3. 

2. Now copy and run the following commands to reset your environment:
```bash
aws cloudformation delete-stack --stack-name 'devlab-s3-bucket'
aws cloudformation delete-stack --stack-name 'devlab-s3-bucket-pipeline'
rm -rf ~/environment/devlab-s3-bucket-pipeline-website
cd ~/environment/devlab-s3-codepipeline
```

3. Close your Cloud9 terminal.

### Survey

Thank you for participating in this lab. Please leave us feedback to let us know how we did and for us to improve in future labs. If the QR code below doesn't work, you can click on the link [here](https://eventbox.dev/survey/S7CF9TH).

![Survey QR Code](/qr/s3-codepipeline.png)


