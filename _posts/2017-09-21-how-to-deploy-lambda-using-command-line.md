---
layout: post
title:  "How To Deploy Lambda Using Command Line"
date:   2017-09-21
---
```sh
#!/bin/sh

## Init.

# Define variables.
source_bucket=asimj-lambda-test
lambda_name=helloworld
lambda_execution_role_name=lambda-${lambda_name}-execution
lambda_execution_access_policy_name=lambda-${lambda_name}-execution-access
lambda_invocation_role_name=lambda-${lambda_name}-invocation
lambda_invocation_access_policy_name=lambda-${lambda_name}-invocation-access
log_group_name=/aws/lambda/${lambda_name}


## Policies.

# Create execution role.
lambda_execution_role_arn=$(aws iam create-role \
  --role-name "$lambda_execution_role_name" \
  --assume-role-policy-document '{
      "Version": "2012-10-17",
      "Statement": [ {
          "Sid": "",
          "Effect": "Allow",
          "Principal": { "Service": "lambda.amazonaws.com" },
          "Action": "sts:AssumeRole" } ] }' \
  --output text \
  --query 'Role.Arn'
)
echo lambda_execution_role_arn=$lambda_execution_role_arn

# Add execution role policy.
aws iam put-role-policy \
  --role-name "$lambda_execution_role_name" \
  --policy-name "$lambda_execution_access_policy_name" \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [ {
        "Effect": "Allow",
        "Action": [ "logs:*" ],
        "Resource": "arn:aws:logs:*:*:*" } ] }'

## Function.

# Create source file.
cat > helloworld.js <<'END' 
console.log('Loading function');
exports.handler = function(event, context, callback) {
    console.log('event=', event)
    var return_value = {"args_received": event}
    callback(null, return_value)
};
END

# Zip code.
zip -r ${lambda_name}.zip ${lambda_name}.js

# Create function.
aws lambda create-function \
  --function-name "${lambda_name}" \
  --zip-file fileb://$PWD/${lambda_name}.zip \
  --role "$lambda_execution_role_arn" \
  --handler "${lambda_name}.handler" \
  --timeout 30 \
  --runtime nodejs4.3

# Update function if you have changed the JS file.
zip -r ${lambda_name}.zip ${lambda_name}.js
cat ${lambda_name}.js
aws lambda update-function-code \
  --function-name "${lambda_name}" \
  --zip-file fileb://$PWD/${lambda_name}.zip 

## Invoke.

# Create payload.
cat > ${lambda_name}-data.json <<'END'
{ "key3": "value3", "key2": "value2", "key1": "value1" }
END

# Invoke function.
aws lambda invoke \
  --function-name "${lambda_name}" \
  --payload file://$PWD/${lambda_name}-data.json \
  ${lambda_name}-output.txt
cat ${lambda_name}-output.txt; echo
 
# Invoke async; returns 202 on success.
aws lambda invoke-async \
  --function-name "${lambda_name}" \
  --invoke-args "${lambda_name}-data.json"

## Logs.

# Describe log groups.
aws logs describe-log-groups \
  --output text \
  --query 'logGroups[*].[logGroupName]'

# Get log stream names.
log_stream_names=$(aws logs describe-log-streams \
  --log-group-name "$log_group_name" \
  --output text \
  --query 'logStreams[*].logStreamName')
echo log_stream_names="'$log_stream_names'"

# View logs.
for log_stream_name in $log_stream_names; do
  aws logs get-log-events \
    --log-group-name "$log_group_name" \
    --log-stream-name "$log_stream_name" \
    --output text \
    --query 'events[*].message'
done | less


## S3 Trigger

# Create bucket
source_bucket=asimj-lambda-test
aws s3 mb s3://${source_bucket}

# Create invocation role.
lambda_invocation_role_arn=$(aws iam create-role \
  --role-name "$lambda_invocation_role_name" \
  --assume-role-policy-document '{
      "Version": "2012-10-17",
      "Statement": [ {
          "Sid": "",
          "Effect": "Allow",
          "Principal": { "Service": "s3.amazonaws.com" },
          "Action": "sts:AssumeRole",
          "Condition": { "StringLike": { "sts:ExternalId": "arn:aws:s3:::*" } } } ] }' \
  --output text \
  --query 'Role.Arn'
)
echo lambda_invocation_role_arn=$lambda_invocation_role_arn

# Get lambda ARN.
lambda_function_arn=$(aws lambda get-function-configuration \
  --function-name "${lambda_name}" \
  --output text \
  --query 'FunctionArn')
echo lambda_function_arn=$lambda_function_arn

# Create invocation role policy.
aws iam put-role-policy \
  --role-name "$lambda_invocation_role_name" \
  --policy-name "$lambda_invocation_access_policy_name" \
  --policy-document '{
     "Version": "2012-10-17",
     "Statement": [ {
         "Effect": "Allow",
         "Action": [ "lambda:InvokeFunction" ],
         "Resource": [ "*" ] } ] }'

# Configure S3 notification.
aws s3api put-bucket-notification \
  --bucket "$source_bucket" \
  --notification-configuration '{
    "CloudFunctionConfiguration": {
      "CloudFunction": "'$lambda_function_arn'",
      "InvocationRole": "'$lambda_invocation_role_arn'",
      "Event": "s3:ObjectCreated:*" } }'

# Upload file.
echo 'hello world' | aws s3 cp - s3://${source_bucket}/1.txt

# View logs.
for log_stream_name in $log_stream_names; do
  aws logs get-log-events \
    --log-group-name "$log_group_name" \
    --log-stream-name "$log_stream_name" \
    --output text \
    --query 'events[*].message'
done | less

## Clean up.

# Delete bucket contents.
aws s3 rm s3://${source_bucket} --recursive

# Delete function.
aws lambda delete-function \
  --function-name "${lambda_name}"

# Delete execution role.
aws iam delete-role-policy \
  --role-name "$lambda_execution_role_name" \
  --policy-name "$lambda_execution_access_policy_name"
aws iam delete-role \
  --role-name "$lambda_execution_role_name"

# Delete invocation role.
aws iam delete-role-policy \
  --role-name "$lambda_invocation_role_name" \
  --policy-name "$lambda_invocation_access_policy_name"
aws iam delete-role \
  --role-name "$lambda_invocation_role_name"

# Delete logs.
log_stream_names=$(aws logs describe-log-streams \
  --log-group-name "$log_group_name" \
  --output text \
  --query 'logStreams[*].logStreamName') &&
for log_stream_name in $log_stream_names; do
  echo "deleting log-stream $log_stream_name"
  aws logs delete-log-stream \
    --log-group-name "$log_group_name" \
    --log-stream-name "$log_stream_name"
done
aws logs delete-log-group \
  --log-group-name "$log_group_name"
```
