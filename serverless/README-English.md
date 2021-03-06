# Amazon S3 MultiThread Resume Migration Serverless Solution  
  
Cluster & Serverless Version 0.94  
  
## Serverless Version  
Amazon EC2 Autoscaling Group Cluster and Serverless AWS Lambda can be deployed together, or seperated used in different senario  
* Transmission between AWS Global and AWS China: Serverless version is suitable for unschedule burst migration.  
* Serverless solution with AWS Lambda and SQS can support large file of tens of GBytes size with unique resumable technique, no worry of 15 mins timeout of Lambda.  
* **Fast and stable**: Multiple nodes X Multiple files/node X Multiple threads/file. Support mass of hugh file concurrently migration. Auto-seperate small file ( including 0 Size) and large file for different procedure.  
* Optional to setup VPC NAT with TCP BBR enable and control EIP to accelerating in some special case. In normal case, e.g. us-east-1, no need for NAT to accelerate.   
* **Reliability**: Amazon SQS queue managed files level jobs, break-point resume trasmission, with timeout protection. Every part will be verified with MD5 after transmission. Single Point of True, final file merging takes the destination S3 as standard, to ensure integrity.  
* **Security**: Transfer in memory, no writing to disk. SSL encryption on transmission. Open source for audit. Leverage IAM role and Lambda Environment Variable(KMS) to store credential Access Key.  
* **Controlable operation**: Job dispatched match the speed of transmission. System capacity predictable. DynamoDB and SQS read/write frequency only related to file numbers, no related to file size. Auto-collect logs to CloudWatch log group. AWS CDK auto deploy.  
* **Elastic cost optimization**: Serverless AWS Lambda only pay for invocation. Support all kind of S3 Storage Class, save long term storage cost.
  
  Serverless Version Architecture  
  
![Serverless Diagram](./img/05.png)  

### Performance Test:
* us-east-1 AWS Lambda ( No need of NAT) 1310 of files, from 4MB to 1.3GB, totally 50GB. Accomplish transmission to China cn-northwest-1 in 10 minutes. Max Lambda concurrency runtime 153. As file size growing up, transmission spent time is no growing up significantly. Each file is handled by one Lambda runtime, bigger file, will be more threads running concurrently in one runtime. In this case, Lambda setup as 1GB mem, according to your job file size, you can optimize to best cost.

![Performance testing](./img/07b.png)  

* ap-northeast-1 AWS Lambda ( Using VPC NAT Instance with EIP and TCP BBR enabled ). Single file of 80GB video file and 45GB zip file. It takes about from 2 to 6 hours to complete transmit to cn-northwest-1. It is running in ONE Lambda concurrency.  

After one AWS Lambda runtime timeout of 15 minutes, SQS message InvisibleTime timeout also. The message recover to Visible, it trigger another Lambda runtime to fetch part number list from dest. Amazon S3 and continue subsequent parts upload. Here is snapshot of logs of Lambda:   

![Big file break-point resume](./img/06.png)  
1. AWS Lambda get dest. Amazon S3 upload id list, select the last time of record
2. Try to get part number list from dest. Amazon S3
3. While network disconnect, auto delay and retry
4. Got the part number list from dest. Amazon S3
5. Auto matching the rest part number 
6. Download and upload the rest parts

### Auto Deploy Dashboard  
![dashboard](./img/09.png)

## CDK auto deploy
Please install AWS CDK and follow instruction to deploy  
https://docs.aws.amazon.com/cdk/latest/guide/getting_started.html   

### 1. Pre-deploy Configure
* Before deploy AWS CDK, please config AWS System Manager Parameter Store manually with this credential:  
Name: s3_migration_credentials   
Type: SecureString  
Tier: Standard  
KMS key source：My current account/alias/aws/ssm  or others you can have  
This s3_migration_credentials is for accessing the account which is not the same as AWS Lambda running account. Config example:  
```
{
  "aws_access_key_id": "your_aws_access_key_id",
  "aws_secret_access_key": "your_aws_secret_access_key",
  "region": "cn-northwest-1"
}
```
Configuration snapshot of s3_migration_credentials:   
![Config snapshot](./img/05.png)  
* Edit app.py in CDK project with your Source and Destination S3 buckets information as example as below:  
```
[{
    "src_bucket": "your_global_bucket_1",
    "src_prefix": "your_prefix",
    "des_bucket": "your_china_bucket_1",
    "des_prefix": "prefix_1"
    }, {
    "src_bucket": "your_global_bucket_2",
    "src_prefix": "your_prefix",
    "des_bucket": "your_china_bucket_2",
    "des_prefix": "prefix_2"
    }]
```
These information will be deployed to System Manager Parameter Store as s3_migration_bucket_para  
* Change your notification email in app.py   
alarm_email = "alarm_your_email@email.com"  

### 2. CDK Auto-deploy
./cdk-serverless  This CDK is written in PYTHON, and will deploy the following resources:  
* Option1: Create S3 Bucket, all new objects in this bucket will be transmitted by Lambda Worker.   
If you want an exist S3 bucekt to trigger SQS/Lambda, you can set the trigger manually. (This is not config in this CDK)
* Option2: If you can setup S3 bucket to trigger SQS/Lambda, e.g. only allow the read access to theses S3 buckets, then this CDK deploy Lambda Jobsender to cron scan these Buckets, compare and create jobs to SQS then trigger Lambda Worker to transmit. You only need to config the buckets information in app.py
* Create Amazon SQS Queue and a related SQS Queue Death Letter Queue. Set InVisibleTime 15 minutes, valid 14 days, 100 Maximum Receives Redrive Policy to DLQ  
* Create Amazon DynamoDB Table for statistic purpose.   
* Upload AWS Lambda code and config environment variable. Config Lambda timeout 15 mintues and 1GB mem. Auto config Lambda role to access the S3, SQS queue and DynamoDB Table under least priviledge.  
There are two Lambda functions, one is Jobsender which is triggered by CloudWatch Cron Event to scan S3 buckets and creat jobs to SQS; The other is Worker, triggered by SQS then to transmit S3 objects.    
* AWS CDK will create a new CloudWatch Dashboard: s3_migrate_serverless to monitor SQS queue and Lambda running status
* Create 3 customized Log Filter for Lambda Log Group. They are to filter Uploading, Downloading, Complete parts Bytes, and statistic them as Lambda traffic and publish to Dashboard. And 2 more filters to catch WARNING and ERROR logs.  
* Ignore List: Jobsender support ignore some objects while comparing bucket. Edit s3_migration_ignore_list.txt in CDK, add the file bucket/key as one file one line. Or use wildcard "*" or "?". E.g.  
```
your_src_bucket/your_exact_key.mp4
your_src_bucket/your_exact_key.*
your_src_bucket/your_*
*.jpg
*/readme.md
```
CDK will deploy this to SSM Parameter Store, you can also change this parameter on s3_migrate_ignore_list after deployed.  

### If Manually Config, notice:  
* Amazon SQS Access Policy to allow Amazon S3 bucket to send message as below, please change the account and bucket in this json:
```json
{
  "Version": "2012-10-17",
  "Id": "arn:aws:sqs:us-east-1:your_account:s3_migrate_file_list/SQSDefaultPolicy",
  "Statement": [
    {
      "Sid": "Sidx",
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "SQS:SendMessage",
      "Resource": "arn:aws:sqs:us-east-1:your_account:s3_migrate_file_list",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::source_bucket"
        }
      }
    }
  ]
}
```
* Create Lambda log group with 3 Log filter, to match below Pattern:
```
Namespace: s3_migrate
Filter name: Downloading-bytes
Pattern: [info, date, sn, p="--->Downloading", bytes, key]
Value: $bytes
default value: 0
Filter name: Uploading-bytes
Pattern: [info, date, sn, p="--->Uploading", bytes, key]
Value: $bytes
default value: 0
Filter name: Complete-bytes
Pattern: [info, date, sn, p="--->Complete", bytes, key]
Value: $bytes
default value: 0
```
So Lambda traffic bytes/min will be emit to CloudWatch Metric for monitoring. Config monitor Statistic: Sum with 1 min.

## Limitation and Notice: 
* Required memory = concurrency * ChunckSize. For the file < 50GB, ChunckSize is 5MB. For the file >50GB, ChunkSize will be auto change to Size/10000  
So for example is file average Size is 500GB , ChunckSize will be auto change to 50MB, and the default concurrency setting 5 File x 10 Concurrency/File = 50. So required 50 x 50MB = 2.5GB memory to run in EC2 or Lambda.  
If you need to increase the concurrency, can change it in config, but remember provision related memory for EC2 or Lambda.  

* This project doesn't support version control, but only get the lastest version of object from S3. Don't change the original file while copying.  
If you change, it will cause the destination file wrong. Workaround is to run the Jobsender(cluster version) after each batch of migration job. Jobsender will compare source and destination s3 file key and size, if different size, it will send jobs to sqs to trigger copy again. But if size is same, it will take it as same file.  

* Don't change the chunksize while start data copying.  

* Jobsender only compare the file Bucket/Key and Size. That means the same filename in the same folder and same size, will be taken as the same by jobsender or single node s3_uploader. If the jobs are trigger by S3 new created object to SQS, then no compare, just copy directly.  

* To delete the resources only need one command: cdk destroy  
But you need to delete these resources manually: DynamoDB, CloudWatch Log Group, new created S3 bucket 


## License

This library is licensed under the MIT-0 License. See the LICENSE file.
