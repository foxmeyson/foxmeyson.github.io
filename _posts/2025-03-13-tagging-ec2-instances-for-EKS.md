---
layout: post
title: Tagging all ec2 instances for all EKSs in account.
subtitle: Lambda, EventBridge and CloudTrail
excerpt_image: /assets/images/posts/tagging-all-ec2-for-EKS/aws-logo-glitch.webp
categories: AWS
tags: [AWS, EKS, ec2]
---

![banner](/assets/images/posts/tagging-all-ec2-for-EKS/aws-logo-glitch.webp)

I recently came across an interesting task. For cost management, it was necessary to integrate all ec2 instances on the AWS account. The tag should contain `Name = EKS-$CLUSTER-NAME`.
As you know, ec2 clusters created for EKS do not have the `Name` tag by default, they are created within the Node Group from a custom Launch Template (if you explicitly specified and created it). Or they are created with an AWS managed Launch Template that controls ec2 in your Node Group, unless you explicitly specified otherwise.
With the first case, when you have a custom Launch Template, everything is clear, you can simply add custom tags to it, and get ec2 instances with `Name = EKS-$CLUSTER-NAME` at the output. But what if some of the Node Group EKS are not managed through a separate Launch Template?
By default, AWS does not have a property that allows you to create an tag for an EKS node and link it to ec2. As a result, when listing ec2 in your account, you can see a huge number of instances without a name, which will only have service tags that EKS uses to manage ec2. One of them is: `kubernetes.io/cluster/$CLUSTER-NAME = owned`, let's try to use it.
I used this tag to assign a custom tag `Name = EKS-$CLUSTER-NAME` for new ec2 instances at the time of their creation, using AWS Lambda and events from EventBridge for this purpose.

## Step 1. Prepare the IAM policy and the IAM role

Create a lambda_ec2_policy policy file.json:

```bash
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:CreateTags"
      ],
      "Resource": "*"
    }
  ]
}
```

Create a policy in AWS:

```bash
aws iam create-policy \
  --policy-name LambdaEC2TaggingPolicy \
  --policy-document file://lambda_ec2_policy.json
```

Create an IAM role for the Lambda function:

```bash
aws iam create-role \
  --role-name LambdaEC2TaggingRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'
```

Link the IAM policy to the role:

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
aws iam attach-role-policy \
  --role-name LambdaEC2TaggingRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/LambdaEC2TaggingPolicy
```

You also need to add a standard policy for logging logs to CloudWatch.:

```bash
aws iam attach-role-policy \
  --role-name LambdaEC2TaggingRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

## Step 2. Creating a Lambda Function

Save your code to a file lambda_function.py:

```python
import boto3
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info(f"Event received: {json.dumps(event)}")
    
    ec2_client = boto3.client('ec2')
    
    instance_ids = extract_instance_ids(event)
    
    if not instance_ids:
        logger.warning("Instance IDs were not found in the event.")
        return {'status': 'no instance ids found'}
    
    logger.info(f"Instance IDs are extracted: {instance_ids}")
    
    tagged_instances = []
    for instance_id in instance_ids:
        if tag_instance(ec2_client, instance_id):
            tagged_instances.append(instance_id)
    
    return {
        'status': 'processed',
        'instances_total': len(instance_ids),
        'instances_tagged': len(tagged_instances),
        'instances': tagged_instances
    }

def extract_instance_ids(event):
    instance_ids = []
    
    if 'detail-type' in event:
        detail_type = event.get('detail-type')
        logger.info(f"Type of event: {detail_type}")
        
        if detail_type == "AWS API Call via CloudTrail":
            try:
                items = event.get('detail', {}).get('responseElements', {}).get('instancesSet', {}).get('items', [])
                logger.info(f"instancesSet elements found: {json.dumps(items)}")
                
                for item in items:
                    instance_id = item.get('instanceId')
                    if instance_id:
                        instance_ids.append(instance_id)
            except Exception as e:
                logger.error(f"Error extracting instance IDs from CloudTrail: {str(e)}")
        
        elif detail_type == "EC2 Instance State-change Notification":
            instance_id = event.get('detail', {}).get('instance-id')
            if instance_id:
                instance_ids.append(instance_id)
    
    elif 'resources' in event:
        resources = event.get('resources', [])
        for resource in resources:
            if resource.startswith('arn:aws:ec2:') and '/instance/' in resource:
                instance_id = resource.split('/instance/')[1]
                instance_ids.append(instance_id)
    
    if not instance_ids and 'instance_id' in event:
        instance_ids.append(event['instance_id'])
    
    if not instance_ids and 'instanceId' in event:
        instance_ids.append(event['instanceId'])
    
    return instance_ids

def tag_instance(ec2_client, instance_id):
    logger.info(f"Instance Processing {instance_id}")
    
    try:
        import time
        time.sleep(30)
        
        retries = 3
        for attempt in range(retries):
            try:
                response = ec2_client.describe_instances(InstanceIds=[instance_id])
                logger.info(f"Received information about the instance on the attempt {attempt+1}")
                break
            except Exception as e:
                if attempt < retries - 1:
                    logger.warning(f"Couldn't get instance data {instance_id}, attempt {attempt+1}: {str(e)}")
                    time.sleep(10 * (attempt + 1))
                else:
                    logger.error(f"Couldn't get instance data {instance_id} adter {retries} attempts: {str(e)}")
                    return False
        
        reservations = response.get('Reservations', [])
        if not reservations or len(reservations[0].get('Instances', [])) == 0:
            logger.warning(f"The instance was not found: {instance_id}")
            return False
        
        instance = reservations[0]['Instances'][0]
        
        instance_tags = {}
        for tag in instance.get('Tags', []):
            instance_tags[tag['Key']] = tag['Value']
        
        logger.info(f"Current instance tags {instance_id}: {json.dumps(instance_tags)}")
        
        if 'Name' in instance_tags:
            logger.info(f"Instance {instance_id} already has the Name tag: {instance_tags['Name']}")
            return False
            
        cluster_name = None
        for key in instance_tags.keys():
            if key.startswith('kubernetes.io/cluster/'):
                cluster_name = key.split('/', 2)[-1]
                logger.info(f"Cluster tag found: {key}={instance_tags[key]}")
                break
        
        if cluster_name:
            logger.info(f"Adding the tag Name=EKS-{cluster_name} for the instance {instance_id}")
            try:
                ec2_client.create_tags(
                    Resources=[instance_id],
                    Tags=[{'Key': 'Name', 'Value': f'EKS-{cluster_name}'}]
                )
                logger.info(f"Tag Name=EKS-{cluster_name} successfully added for instance {instance_id}")
                return True
            except Exception as e:
                logger.error(f"Error when creating the tag: {str(e)}")
                return False
        else:
            logger.info(f"For the instance {instance_id} cluster EKS tags not found")
            return False
            
    except Exception as e:
        logger.error(f"Unexpected error during instance processing {instance_id}: {str(e)}")
        return False
```

Create a ZIP archive with the function:

```bash
zip lambda_function.zip lambda_function.py
```

Create a Lambda function in the AWS CLI:

```bash
aws lambda create-function \
  --function-name ec2-auto-tagging \
  --runtime python3.11 \
  --zip-file fileb://lambda_function.zip \
  --handler lambda_function.lambda_handler \
  --role arn:aws:iam::${ACCOUNT_ID}:role/LambdaEC2TaggingRole \
  --timeout 60
```

## Step 3. Configure EventBridge and CloudTrail

Create an EventBridge rule that triggers Lambda when creating new EC2 instances.:

```bash
aws events put-rule \
  --name "trigger-on-ec2-instance-creation" \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
      "eventSource": ["ec2.amazonaws.com"],
      "eventName": ["RunInstances"]
    }
  }'
```

### Configuring CloudTrail to record API events

CloudTrail is necessary for the system to receive instance startup events. If CloudTrail is not configured, follow these steps:

```bash
export S3_BUCKET=cloudtrail-logs-$(aws sts get-caller-identity --query "Account" --output text)
export ACCOUNT_ID=$(aws sts get-caller-identity --query "Account" --output text)
export REGION=$(aws configure get region)

aws s3 mb s3://$S3_BUCKET --region $REGION

cat > bucket-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AWSCloudTrailAclCheck",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::${S3_BUCKET}"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::${S3_BUCKET}/AWSLogs/${ACCOUNT_ID}/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        }
    ]
}
EOF

aws s3api put-bucket-policy --bucket $S3_BUCKET --policy file://bucket-policy.json

aws cloudtrail create-trail \
  --name api-events-trail \
  --s3-bucket-name $S3_BUCKET \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name api-events-trail

aws cloudtrail put-event-selectors \
  --trail-name api-events-trail \
  --event-selectors '[{"ReadWriteType": "All", "IncludeManagementEvents": true}]'
```

### Add the Lambda permission and configure the EventBridge

```bash
aws lambda add-permission \
  --function-name ec2-auto-tagging \
  --statement-id AllowEventBridgeInvoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn arn:aws:events:${REGION}:${ACCOUNT_ID}:rule/trigger-on-ec2-instance-creation

aws events put-targets \
  --rule trigger-on-ec2-instance-creation \
  --targets '[{"Id": "1", "Arn": "arn:aws:lambda:'${REGION}':'${ACCOUNT_ID}':function:ec2-auto-tagging"}]'
```

## Step 4. Checking the work

Create a new EC2 instance with a tag like:

```bash
aws ec2 run-instances \
  --image-id ami-XXXXXXXXXX \
  --count 1 \
  --instance-type t2.micro \
  --subnet-id subnet-XXXXXXXXXX \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=kubernetes.io/cluster/my-cluster,Value=owned}]'
```

After starting the instance, make sure that a new tag is automatically added:

```bash
INSTANCE_ID=i-XXXXXXXXXX

sleep 60

aws ec2 describe-tags --filters "Name=resource-id,Values=${INSTANCE_ID}" --query "Tags[?Key=='Name']"
```

The tag should appear:

```json
[
  {
    "Key": "Name",
    "ResourceId": "i-XXXXXXXXXX",
    "ResourceType": "instance",
    "Value": "EKS-my-cluster"
  }
]
```

## Step 5. Tag existing ec2 instances

After all the actions above, at the time of creation, an tag with the cluster name will be created on the instance, but right now there are instances created without tags, let's tag them too:

```bash
#!/bin/bash

set -e

if ! command -v aws &> /dev/null; then
    echo "Error: AWS CLI is not installed. Please install it and configure it."
    exit 1
fi

echo "Getting a list of all EKS clusters from existing EC2 instances..."
CLUSTER_NAMES=($(aws ec2 describe-instances \
    --filters "Name=tag-key,Values=aws:eks:cluster-name" \
    --query "Reservations[].Instances[].Tags[?Key=='aws:eks:cluster-name'].Value" \
    --output text | sort | uniq))

if [ ${#CLUSTER_NAMES[@]} -eq 0 ]; then
    echo "No instances were found with the aws:eks:cluster-name tag."
    exit 0
fi

echo "The following EKS clusters were found:"
for CLUSTER in "${CLUSTER_NAMES[@]}"; do
    echo "- $CLUSTER"
done

TOTAL_INSTANCES=0

for CLUSTER_NAME in "${CLUSTER_NAMES[@]}"; do
    NAME_TAG_VALUE="EKS_${CLUSTER_NAME}"
    
    echo "========================================"
    echo "Processing cluster: ${CLUSTER_NAME}"
    echo "Searching for EC2 instances with the aws tag:eks:cluster-name=${CLUSTER_NAME}..."
    
    INSTANCE_IDS=$(aws ec2 describe-instances \
        --filters "Name=tag:aws:eks:cluster-name,Values=${CLUSTER_NAME}" \
        --query "Reservations[].Instances[].InstanceId" \
        --output text)
    
    if [ -z "$INSTANCE_IDS" ]; then
        echo "Instances with the tag aws:eks:cluster-name=${CLUSTER_NAME} not found."
        continue
    fi
    
    INSTANCE_COUNT=$(echo $INSTANCE_IDS | wc -w)
    echo "Found $INSTANCE_COUNT instances for the cluster ${CLUSTER_NAME}."
    TOTAL_INSTANCES=$((TOTAL_INSTANCES + INSTANCE_COUNT))
    
    for INSTANCE_ID in $INSTANCE_IDS; do
        echo "Adding a tag Name=${NAME_TAG_VALUE} to the instance $INSTANCE_ID..."
        
        EXISTING_NAME_TAG=$(aws ec2 describe-tags \
            --filters "Name=resource-id,Values=${INSTANCE_ID}" "Name=key,Values=Name" \
            --query "Tags[0].Value" \
            --output text)
        
        if [ "$EXISTING_NAME_TAG" != "None" ] && [ ! -z "$EXISTING_NAME_TAG" ]; then
            echo " The instance already has the tag  Name=${EXISTING_NAME_TAG}. Updating it..."
        fi
        
        aws ec2 create-tags \
            --resources "$INSTANCE_ID" \
            --tags "Key=Name,Value=${NAME_TAG_VALUE}"
        
        echo " The tag has been successfully added."
    done
    
    echo "Processing of the cluster ${CLUSTER_NAME} has been completed."
done

echo "========================================"
echo "The operation is completed. A total of $TOTAL_INSTANCES instances were processed across all clusters."
```

## Debugging when problems occur

If the tags do not appear automatically, follow these steps for debugging:

### 1. Checking Lambda logs

```bash
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name /aws/lambda/ec2-auto-tagging \
  --order-by LastEventTime \
  --descending \
  --limit 1 \
  --query 'logStreams[0].logStreamName' \
  --output text)

aws logs get-log-events \
  --log-group-name /aws/lambda/ec2-auto-tagging \
  --log-stream-name $LOG_STREAM \
  --limit 20
```

### 2. Checking EventBridge settings

```bash
aws events describe-rule --name trigger-on-ec2-instance-creation

aws events list-targets-by-rule --rule trigger-on-ec2-instance-creation
```

### 3. Checking CloudTrail

```bash
aws cloudtrail get-trail-status --name api-events-trail

aws cloudtrail get-event-selectors --trail-name api-events-trail
```

### 4. Verifying the rights of the IAM role

```bash
aws iam list-attached-role-policies --role-name LambdaEC2TaggingRole
```

### 5. Manual Lambda testing with test event

```bash
cat > test-event.json << EOF
{
  "version": "0",
  "id": "6a7e8feb-b491-4cf7-a9f1-bf3703467718",
  "detail-type": "AWS API Call via CloudTrail",
  "source": "aws.ec2",
  "account": "$(aws sts get-caller-identity --query "Account" --output text)",
  "time": "2021-12-03T17:31:20Z",
  "region": "$(aws configure get region)",
  "resources": [],
  "detail": {
    "eventSource": "ec2.amazonaws.com",
    "eventName": "RunInstances",
    "responseElements": {
      "instancesSet": {
        "items": [
          {
            "instanceId": "i-YOUR_INSTANCE_ID"
          }
        ]
      }
    }
  }
}
EOF

INSTANCE_ID=i-XXXXXXXXXX
sed -i "s/i-YOUR_INSTANCE_ID/$INSTANCE_ID/g" test-event.json

aws lambda invoke \
  --function-name ec2-auto-tagging \
  --payload fileb://test-event.json \
  response.json

cat response.json
```

### 6. The main causes of inactivity and their solutions

EventBridge does not have Lambda as a target

```bash
aws events put-targets \
    --rule trigger-on-ec2-instance-creation \
    --targets '[{"Id": "1", "Arn": "arn:aws:lambda:'${REGION}':'${ACCOUNT_ID}':function:ec2-auto-tagging"}]'
```

Lambda does not have permission to receive events from EventBridge

```bash
aws lambda add-permission \
    --function-name ec2-auto-tagging \
    --statement-id AllowEventBridgeInvoke \
    --action lambda:InvokeFunction \
    --principal events.amazonaws.com \
    --source-arn arn:aws:events:${REGION}:${ACCOUNT_ID}:rule/trigger-on-ec2-instance-creation
```

CloudTrail is not configured or enabled

```bash
aws cloudtrail start-logging --name api-events-trail
```

The Lambda function shuts down too quickly, without waiting for the tags to become available.

```bash
aws lambda update-function-configuration \
    --function-name ec2-auto-tagging \
    --timeout 120
```
