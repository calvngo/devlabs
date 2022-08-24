# devlab-serverless-sentiment-analysis


* Estimated time to complete this lab: 15 minutes
* Learning Outcomes: Build serverless data pipeline to process unstructured data (text) in your data lake and generate sentiment scores and labels.
* Lab level: L300



# AWS services leveraged
AWS Glue is a serverless data integration service that makes it easy to discover, prepare, and combine data for analytics, machine learning, and application development.

Amazon Comprehend is a natural-language processing (NLP) service that uses machine learning to uncover valuable insights and connections in text. You can use Amazon Comprehend to identify the language of the text, extract key phrases, places, people, brands, or events, understand sentiment about products or services, and identify the main topics from a library of documents

AWS Lambda is a serverless, event-driven compute service that lets you run code for virtually any type of application or backend service without provisioning or managing servers.

AWS Step Functions is a low-code, visual workflow service that developers use to build distributed applications, automate IT and business processes, and build data and machine learning pipelines using AWS services.

Amazon Athena is an interactive query service that makes it easy to analyze data in Amazon S3 using standard SQL. Athena is serverless, so there is no infrastructure to manage, and you pay only for the queries that you run.

Amazon QuickSight is a cloud-scale business intelligence (BI) service that you can use to deliver easy-to-understand insights to the people who you work with, wherever they are.


# Solution Overview
The below diagram illustrates the solution architecture. AWS Step Function is used to orchestrate end to end data pipeline. As part of the data pipeline, the first AWS Glue job makes an API call to Amazon Comprehend service and waits for the Comprehend job to finish. Amazon Comprehend reads data from Amazon S3, generates sentiment scores for each record and then writes back the processed dato to S3 in csv format. The second Glue job converts csv to parquet format, which is an efficient columnar file format for analytics.  
      Amazon Athena is then used to create table structure on top of optimised parquet data and run analytical queries . Please note AWS Glue maintains the database catalogue and database table structure while Amazon Athena queries data in Amazon S3 using the AWS Glue database catalogue. The dataset is made of text description from movie reviews(source imdb). Finally we will visualise the results using Amazon QuickSight


![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/arch_diagram.jpg)

As part of this lab, you will be performing below high-level tasks:

* Reset environment by running commands using AWS CLI. This step creates a new bucket in Amazon S3, uploads raw data and AWS Glue scripts in S3, and executes AWS Cloud Formation script to create AWS Step Function and AWS Glue jobs.  
* Trigger AWS Step Function to process raw data (text file) and generate sentiment analysis results in parquet format
* Create table in Amazon Athena, which points to processed/optimised data
* Visualise the results using Amazon QuickSight
* Terminate environment by running commands using AWS CLI



First, login to AWS Console. Choose **us-west-2 region**. For this lab, we'll use the AWS Cloud 9 web-based IDE to run few CLI commands to reset environment

# Step 1 : Reset environment. 

* Navigate to the AWS Cloud 9 web console. To access Cloud9 search for Cloud9 in the AWS Console and Click on Cloud9.
![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/Cloud9.png)  
* Then select the Cloud9 environment named `Serverless Sentiment Analysis (Level 300)', and click the 'Open IDE' button for you to enter the IDE. 

![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/ide.png)  
* After the interface loads, find the tab with the Terminal and click to maximize it

* Copy the commands below

```
sudo yum install jq
export AWS_DEFAULT_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
account_id=$(aws sts get-caller-identity --query 'Account' --output text)
dataset_list=$(aws quicksight list-data-sets --aws-account-id $account_id --query 'DataSetSummaries[?Name ==`movie_review_score`].DataSetId' --output text)
for dataset in $dataset_list; do aws quicksight delete-data-set --aws-account-id $account_id --data-set-id $dataset ; done
aws cloudformation delete-stack --stack-name summit-lab
sleep 20
aws glue delete-table --database-name default --name movie_review_score
rm -f  devlab-serverless-sentiment-analysis --recursive 


git clone https://github.com/aws-samples/devlab-serverless-sentiment-analysis
export BUCKET_NAME="devlab-dp-serverless-$(uuidgen)"
aws s3 mb s3://$BUCKET_NAME
aws s3 cp /home/ec2-user/environment/devlab-serverless-sentiment-analysis/artefact s3://$BUCKET_NAME --recursive
sleep 10
aws cloudformation create-stack --stack-name summit-lab --template-url http://s3.amazonaws.com/$BUCKET_NAME/cfn-pipeline.yaml --parameters ParameterKey=datalakebucket,ParameterValue=$BUCKET_NAME --capabilities CAPABILITY_NAMED_IAM
sleep 40
echo 'datalakebucket'=$BUCKET_NAME


```

* In your Cloud9 terminal paste the commands and run it. This step takes about 2 minutes to complete. Wait until you see this statement completing `echo 'datalakebucket'=$BUCKET_NAME` . Also, please ignore this exception as part of reset environment : **Table movie_review_score not found**

* Please make note of Amazon S3 bucket _"datalakebucket"_ created in this step. This will be used in Athena section while creating DDL and in Quicksight section when granting access permission.

Running above commands does following:   
  * Cleans up old files
  * Creates a new bucket in Amazon S3
  * Uploads Amazon S3 files - raw data , AWS Glue scripts
  * Runs AWS Cloud Formation to create AWS Step Function and AWS Glue jobs
     


# Step 2 : Run data pipeline
* Navigate to AWS Step Function in AWS console. Please ensure that you are logged in us-west-2 region.  
* Click on "pipeline-sentiment-analysis" to navigate to below page.
* Click on Start Execution at the top to trigger the pipeline. 
* When prompted on next page, click on "Start Execution" again.

![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/step_function.png)  


The pipeline takes about 8-9 minutes to finish. In the meantime, lets create a table in Amazon Athena that points to optimised data and following that, lets setup Amazon QuickSight. Once the pipeline completes, you will just need to refresh the page in Amazon QuickSight to get results.



# Step 3 : Create DDL using Amazon Athena
* Navigate to Amazon Athena in AWS console. Please ensure that you are logged in us-west-2 region.
* If it’s the first time you are using Athena in your AWS Account, you have to set up S3 bucket for saving results.
  - Click **Explore the query editor**. 
  ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/athena1.png)  
  - Then click on **View settings**. 
  ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/athena2.png)  
  - Next click on **Manage**. 
  ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/athena3.png)  
  - In the pop-up window in the Location of query result field, click on **Browse S3**.  
  ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/athena4.png) 
  - Choose the bucket with that was created earlier during AWS CLI run, then click on Select button. 
  - Click on Save.
  - Return to the **Editor** tab. 


* Before we can perform our pipeline analysis, we need to create the below DDL. Replace the _"datalakebucket"_ part of the LOCATION clause with the  bucket name value captured earlier during AWS CLI run. 

```
      
CREATE EXTERNAL TABLE `default.movie_review_score`(
  `category` string, 
  `movie_name` string, 
  `review_text` string, 
  `line` int, 
  `sentiment` string, 
  `mixed_score` decimal(10,2), 
  `negative_score` decimal(10,2), 
  `neutral_score` decimal(10,2), 
  `positive_score` decimal(10,2))
ROW FORMAT SERDE 
  'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
LOCATION
  's3://<datalakebucket>/optimised/'
 
```
* Execute DDL in Amazon Athena query editor by hitting Run. 
  ![main_arch](./images/athena5.png) 

 
# Step 4: Visualize data using Amazon QuickSight
We can visualize Amazon Comprehend’s sentiment results by using Amazon QuickSight. 

* In the AWS services console, search for QuickSight. Please ensure that you are logged in us-west-2 (Oregon) region. You can check this in the top right corner of the AWS console. 
![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/qs-1.png)
* If this is the first time you have used QuickSight in this account, you are prompted to create an account. When prompted **Click Sign up for QuickSight**. (Note: _If you are a retruning user or do not get prompted to set up account, then we must grant Amazon QuickSight access to Amazon Athena and the associated S3 bucket "datalakebucket" created in this lab. For more information on doing this, please see section "Make sure that you authorized Amazon QuickSight to use Athena" in   https://docs.aws.amazon.com/quicksight/latest/user/troubleshoot-connect-athena.html . Once S3 bucket access is granted, you can jump straight to step **On the top right corner, click New analysis** below._ )
![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/qs-2.png)
* For account type, choose the default **Enterprise** Version. Click **Continue**.
![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/qs-3.png)
* On the Create your QuickSight account page, for QuickSight account name give a unique name (e.g., quicksight-lab-_initals_-_randomstring_) and email address.
* Choose the appropriate AWS region (us-west-2 for this workshop) and the check boxes to enable auto discovery, Amazon Athena, and Amazon S3.
* Under Amazon S3, click **Choose S3 buckets**. Select newly created S3 bucket (through AWS CLI earlier) _"datalakebucket"_ .  
* On next page, Click Finish.
![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/qs-5.png)


* On the top right corner, click **New analysis**. 
![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/qs-6.png)
* Click **New Data Set**.
![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/qs-7.png)
* On the **Create a Dataset** page, select **Athena** as the data source.  
      ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/qs-8.png)
      
* Enter a name for your Athena data source. Lets call it **movie-review-dataset** , validate and choose **Create data source**. 
      ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/validate_dataset.png)
      
* Choose database "default" and table "movie_review_score" created earlier in Amazon Athena. Click **Select**. If you dont see the database or table, then please check if you are in the right region (us-west-2 for this workshop).
      ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/select_table.png)
      
* Choose **Directly query your data** and click **Visualize**. 
      ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/direct_query.png)
      
* Now we can create some visualizations by adding few columns such as "category", "positive_score" and "negative_score" into visualization. You will not see any data if the data pipeline has not completed.  
      ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/visualise.png)      

* Let's open a new tab and check the status of data pipeline by navigating to AWS Step Functions. Once the data pipeline has completed successfully, navigate back to Amazon Quicksight tab and refresh screen. You will be able to see results now.  
            ![main_arch](https://github.com/aws-samples/devlab-serverless-sentiment-analysis/blob/main/images/final_results.png)  
 
 

# Step 5:  Clean up the stack

Return to Cloud9 terminal and execute below commands in terminal window of AWS CLI
```
account_id=$(aws sts get-caller-identity --query 'Account' --output text)
dataset_list=$(aws quicksight list-data-sets --aws-account-id $account_id --query 'DataSetSummaries[?Name ==`movie_review_score`].DataSetId' --output text)
for dataset in $dataset_list; do aws quicksight delete-data-set --aws-account-id $account_id --data-set-id $dataset ; done
aws cloudformation delete-stack --stack-name summit-lab
sleep 20
aws glue delete-table --database-name default --name movie_review_score
rm -f  devlab-serverless-sentiment-analysis --recursive 
aws s3 rb s3://$BUCKET_NAME --force
echo "done"
```

# Lab Survey
We would really appreciate if you can complete the lab survey (link below). This will help us improve your lab experience next time :) 

![qr_code](/qr/serverless-data-pipeline.png)  

[Survey Link](	
https://eventbox.dev/survey/2QBDWOA)
