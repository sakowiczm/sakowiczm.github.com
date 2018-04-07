--- 
layout: post
title: AWS Lambda - how to add S3 trigger using command line
comments: true
date: 2018-04-07
---

Adding new trigger to Lambda function through AWS CLI is two step operation. First we need to grant S3 execution rights to our Lambda function and then configure S3 notification itself.
 
To add permissions, we can use following:

``` bash
aws lambda add-permission `
--function-name aws-lambda-function-name  `
--action lambda:InvokeFunction `
--principal s3.amazonaws.com `
--source-arn arn-of-s3-bucket `
--statement-id 1
``` 

The `--statement-id` (SID) - in this case is just unique identifier of policy and it can be anything. Additionally, following commands will allow us to list and remove policies attached to our function:

``` bash
aws lambda get-policy --function-name aws-lambda-function-name

aws lambda remove-permission --function-name aws-lambda-function-name --statement-id 1
``` 

If there is no policy attached we get `ResourceNotFoundException`.

To configure S3 notification we use following command:

``` bash
aws s3api put-bucket-notification-configuration `
--bucket aws-s3-bucket-name `
--notification-configuration file://notification.json
``` 

`Notification.json` describes how the trigger should look like, for example:

``` json
{
"LambdaFunctionConfigurations": [
    {
      "Id": "my-lambda-function-s3-event-configuration",
      "LambdaFunctionArn": "arn-of-aws-lambda-function",
      "Events": [ "s3:ObjectCreated:Put" ],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "suffix",
              "Value": ".zip"
            }
          ]
        }
      }
    }
  ]
}
``` 

If something is not working as it should be, you can add `--debug` flag to get verbose output from each of the commands.

Documentation:
*	[Adding permissions.](https://docs.aws.amazon.com/lambda/latest/dg/access-control-resource-based.html)
*	[Configure notifications.](https://docs.aws.amazon.com/cli/latest/reference/s3api/put-bucket-notification-configuration.html)

