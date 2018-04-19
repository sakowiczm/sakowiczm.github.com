--- 
layout: post
title: AWS CLI - working with IAM roles
comments: true
date: 2018-04-18
---

This is quick 'remind me' post. To [create role][1] use following command:

``` bash
aws iam create-role --role-name Test-Role --assume-role-policy-document file://role.json
```

Below `role.json` is [trust relationship document][2] that tells AWS that newly created entity assumes role of AWS Lambda.

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

To make role usable we need to [attach policies][3], in this case I attach inline policy:

``` bash
aws iam put-role-policy --role-name Test-Role --policy-name ExamplePolicy --policy-document file://policy.json
```

Here in `policy.json` we are describing what action are allowed for our new role. In this case
our Lambda can interact with DynamoDB and CloudWatch.

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:DescribeStream",
                "dynamodb:GetRecords",
                "dynamodb:GetShardIterator",
                "dynamodb:ListStreams",
                "dynamodb:PutItem",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

To delete role first we need to remove all policies attached. 
We can [list][4] policies attached to role using:

``` bash
aws iam list-role-policies --role-name Test-Role
```
To [remove policy][5]:

``` bash
aws iam delete-role-policy --role-name Test-Role --policy-name ExamplePolicy
```

Finally to [delete role][6]:

``` bash
aws iam delete-role --role-name Test-Role
```

[1]: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-user.html
[2]: https://docs.aws.amazon.com/cli/latest/reference/iam/create-role.html
[3]: https://docs.aws.amazon.com/cli/latest/reference/iam/put-role-policy.html
[4]: https://docs.aws.amazon.com/cli/latest/reference/iam/list-role-policies.html
[5]: https://docs.aws.amazon.com/cli/latest/reference/iam/delete-role-policy.html
[6]: https://docs.aws.amazon.com/cli/latest/reference/iam/delete-role.html
