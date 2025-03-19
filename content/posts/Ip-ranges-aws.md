+++
draft = false
date = 2025-03-19T09:07:38+01:00
title = "Notification on AWS IP range changes"
description = "Getting notified when AWS ip range will change"
slug = ""
authors = ["Fabrizio Regalli"]
tags = ["aws", "sns", "eventbridge", "s3"]
categories = []
externalLink = ""
series = []
disableShare = false
+++



# Notifications for AWS IP Range Changes

There is a need to whitelist the AWS IP range for the sns service on the firewall so that SNS can reach the endpoint present on various servers external to AWS

AWS already provide a way to get notified when ips will be changed, subscribing directly to the SNS "AmazonIpSpaceChanged" Topic. You can find additional information [here](https://docs.aws.amazon.com/vpc/latest/userguide/subscribe-notifications.html)

The limitation of this subscription is that you can't choose the region or even the service: you will be notified for every changes. But what if you are interested only in getting notified when an ip changes in a specific region or service?
There is already a great [website](https://awsiprange.com/) that can helps but we prefer to create a specific "home-made" solution.

To achieve this goal I chose to use four AWS services: sns, s3, lambda and event bridge.

 1. sns: to get notified 
 2. s3: to store ip range file in text format
 3. lambda: a python function that can get the ip list 
 4. event bridge: to schedule the execution of the lambda

**Let's do it**

First, create a new SNS topic to publish IP changes from Lambda function

![snstopic](/images/topic.png)

Subscribe to the SNS topic created previously. I did the Email subscription and the Lambda also (you need to do the lamba subscription later, once the Lambda will be created)

![subsbcription](/images/subscription.png)

Create an S3 bucket. In my case, I created a “aws-ips-eu-central-1” S3 bucket I'm looking for SNS IP ranges in eu-central-1 (Frankfurt) region. Hence the text file contains only the IPs for eu-central-1 from the txt file.

Create a new IAM role to enable lambda function to access S3 and SNS services. The process is explained in the following link https://aws.amazon.com/premiumsupport/knowledge-center/lambda-execution-role-s3-bucket/. Here is how my policy looks like for the role.

        {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListAllMyBuckets",
                    "s3:GetBucketLocation",
                    "s3:ListBucket"
                ],
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": "s3:*",
                "Resource": [
                    "arn:aws:s3:::aws-ips-eu-central-1",
                    "arn:aws:s3:::aws-ips-eu-central-1/*"
                ]
            },
            {
                "Action": [
                    "SNS:Publish",
                    "SNS:Subscribe"
                ],
                "Effect": "Allow",
                "Resource": [
                    "arn:aws:sns:eu-central-1:<accountID>:<ARN of sns topic>"
                ]
            }
        ]
    }

Replace accountID and ARN of sns topic with your real value and attach it to the Lambda.

It's time to create the Lambda function, code is there:

    import json
    import urllib.request
    import boto3
    
    
    def  get_current_ip_ranges():
    url =  "https://ip-ranges.amazonaws.com/ip-ranges.json"
    response = urllib.request.urlopen(url)
    data = json.loads(response.read().decode("utf-8"))
    return data
      
    def  get_stored_ip_ranges(s3_bucket):
    s3 = boto3.client("s3")
    try:
    response = s3.list_objects_v2(Bucket=s3_bucket, Prefix="aws-ip-ranges.txt")
    if  'Contents'  not  in response:
    return  None
    obj = s3.get_object(Bucket=s3_bucket, Key="aws-ip-ranges.txt")
    return  set(obj['Body'].read().decode("utf-8").splitlines())
    except s3.exceptions.NoSuchKey:
    return  None
      
    
    def  store_ip_ranges(s3_bucket, ip_list):
    s3 = boto3.client("s3")
    s3.put_object(Bucket=s3_bucket, Key="aws-ip-ranges.txt", Body="\n".join(ip_list)
    
    def  notify_change(message, topic_arn):
    sns = boto3.client("sns")
    sns.publish(TopicArn=topic_arn, Message=message, Subject="AWS IP Ranges Changed")
    
    def  filter_region_ip_ranges(ip_data, region):
    return {prefix["ip_prefix"] for prefix in ip_data["prefixes"] if prefix.get("region") == region}
      
    
    def  lambda_handler(event, context):
    S3_BUCKET =  "<bucket_name>"
    SNS_TOPIC_ARN =  "arn:aws:sns:eu-central-1:<accountID>:<topic_name>"
    TARGET_REGION =  "eu-central-1"
    
    current_data = get_current_ip_ranges()
    current_filtered = filter_region_ip_ranges(current_data, TARGET_REGION)
    stored_filtered = get_stored_ip_ranges(S3_BUCKET) or  set()
    added_ips = current_filtered - stored_filtered
    removed_ips = stored_filtered - current_filtered
    if added_ips or removed_ips:
    
    message =  f"AWS IP Ranges for {TARGET_REGION} have changed!\n"
    if added_ips:
    message +=  "Added:\n"  +  "\n".join(added_ips) +  "\n"
    if removed_ips:
    message +=  "Removed:\n"  +  "\n".join(removed_ips)
    notify_change(message, SNS_TOPIC_ARN)
    
    store_ip_ranges(S3_BUCKET, current_filtered)
    
    return {"statusCode": 200, "body": "Execution completed successfully."}

Remeber to change bucket_name, accountID, and topic_name accordingly also there.

Subscribe to the Lambda 

![lambdasubscription](/images/lambdasubscription.png)

Create a schedule inside EventBridge: for my needed I used a cron expression to run the Lambda every 10 mins.

    0/10 * * * ? * 

![eventbridge](/images/eventbridge.png)

![eventbridgeschedule](/images/eventbridgeschedule.png)

If everything works correctly, you should receive a mail like this once the ip range for the eu-central-1 region will be updated 

![mailsns](/images/mailsns.png)
