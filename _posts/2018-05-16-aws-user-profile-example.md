--- 
layout: post
title: AWS user profile example
comments: true
date: 2018-05-16
---

This post describes imaginary scenario I implemented to explore parts of AWS with .NET Core. It's not perfect certain parts are left lacking. The idea was to get required steps and find possible pitfalls, not on getting things right. So after you've been warned let's have a look what we are going to build:

![](/img/posts/2018/2018-05-15-schema.png){: .center-image .img-responsive }

Scenario is fairly typical - we have user profile data stored in DocumentDB and we want to allow users to upload their pictures. Because we need to standardize image sizes we also create thumbnails and store it alongside original file. So:

1. User uploads image to S3 with necessary metadata (user id)
2. S3 triggers Lambda function
3. Lambda function - extracts id form S3 object metadata, locates necessary record in DynamoDB and stores image url
4. DynamoDB Stream triggers second Lambda that obtains image from S3 and
5. Passes it to external service to create thumbnail image
6. Resized picture is stored back in S3

We need to be aware that first and last step will trigger the same event - so we need to add proper filters - otherwise we execute our first lambda for new thumbnail image. This will create 'forever' loop and AWS will happily execute whole cycle again and again.

Initially I wanted to resize images using purely C# but unfortunately is not as simple as it sounds. .NET Core still has limited image processing capabilities and all open source libraries I've found were pre-release and plagued with issues. Ultimately I've decide to use external service [TinyPNG.com][1]. If you want to follow along, you have to [register][2] to get your own api key - it's free and it takes just a minute. Following should be fairly simple and require only occasional tweaks - mainly to specify your own resource url/names. All necessary files can be found [here][3]. We start by creating data stores.

##### S3 Bucket

Bucket name need to be unique - so you have to pick your own. To create it we execute:

``` 
aws s3api create-bucket `
    --bucket user-profile-images-01234 `
    --region eu-west-1 `
    --create-bucket-configuration LocationConstraint=eu-west-1
```

When bucket is created we want to open it to the public so users can read, write and delete object. We do this by adjusting its policy:

```
aws s3api put-bucket-policy `
    --bucket user-profile-images-01234 `
    --policy file://bucket-policy.json
```

Policy file looks as below, `Principal` defined as * means everyone, `Actions` describe what is allowed, and `Resource` specifies our newly created bucket.

``` json
{
    "Version":"2012-10-17",
    "Statement":[{
    "Sid":"AllowPublicRead",
        "Effect":"Allow",
        "Principal": "*",
        "Action":["s3:GetObject", "s3:PutObject", "s3:DeleteObject" ],
        "Resource":["arn:aws:s3:::user-profile-images-01234/*"]
    }]
}
```

To test the bucket we can use following commands to upload, list and delete image:

```
aws s3 cp .\maverick.jpg s3://user-profile-images-01234/maverick.jpg 
aws s3 ls s3://user-profile-images-01234
aws s3 rm s3://user-profile-images-01234/maverick.jpg
```

##### DynamoDB

To create table execute:

```
aws dynamodb create-table `
--table-name user-profile `
--attribute-definitions AttributeName=id,AttributeType=N `
--key-schema AttributeName=id,KeyType=HASH `
--provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 `
--stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

We can load data and query table using:

```
aws dynamodb batch-write-item --request-items file://user-profile.json

aws dynamodb scan --table-name user-profile
```

##### Lambdas

To build our functions I'll be using template provided by Amazon - you have to install them if you haven't done this already.

```
dotnet new -i Amazon.Lambda.Templates::*

dotnet new lambda.EmptyFunction `
    --name SetProfileImage `
    --region eu-west-1
```

Before function can be deployed we need to create its execution role and set necessary policies.

```
aws iam create-role `
    --role-name SetProfileImageRole `
    --assume-role-policy-document file://role.json
```

Role document looks like this, and it's fairly standard:

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

Role policy describes what our function will be allowed to do:

```
aws iam put-role-policy `
    --role-name SetProfileImageRole `
    --policy-name SetProfileImageRoleInlinePolicy `
    --policy-document file://set_profile_image_policy.json
```

We allow our lambda to manipulate DynamoDB database and CloudWatch logs. We don't need to add access to S3 as we gave it public access.
In this example I'm not constraining access to a specific resource - for sake of simplicity - this function will have access to all my DynamoDB tables. 
Policy document looks as follows:

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:DescribeTable",
                "dynamodb:GetItem",
                "logs:CreateLogStream",
                "logs:CreateLogGroup",
                "logs:PutLogEvents"
            ],
            "Resource": "*"
        }
    ]
}
```

Now function can be build, package and deploy:

```
dotnet restore
dotnet build
dotnet lambda package

dotnet lambda deploy-function SetProfileImage `
    --function-role SetProfileImageRole `
    --region eu-west-1
```

Because we haven't modified template code - we can easily test it by passing simple string. We should receive back capitalized string: 

```
dotnet lambda invoke-function `
    -fn SetProfileImage `
    --payload "Just Checking If Everything is OK" `
    --region eu-west-1
```

If everything is working as described we can modify our code. First let's install necessary nuget packages:

```
dotnet add package AWSSDK.S3
dotnet add package Amazon.Lambda.S3Events
dotnet add package AWSSDK.DynamoDBv2
```

Finish code will look something like this:

``` csharp
namespace SetProfileImage
{
    public class Function
    {
        private readonly AmazonS3Client _s3client;

        private string metadataKey = "x-amz-meta-user-profile-id";

        public Function()
        {
            _s3client = new AmazonS3Client();
        }
        
        public async Task FunctionHandler(S3Event @event, ILambdaContext context)
        {
            foreach(var record in @event.Records)
            {
                var metadata = await _s3client.GetObjectMetadataAsync(record.S3.Bucket.Name, record.S3.Object.Key);

                if(metadata.Metadata.Keys.Contains(metadataKey))
                {
                    var value = Convert.ToInt32(metadata.Metadata[metadataKey]);

                    await UpdateDatabase(value, record.S3.Object.Key);
                }
            }
        }

        public async Task UpdateDatabase(int id, string fileName)
        {
            IAmazonDynamoDB client = new AmazonDynamoDBClient();
            DynamoDBContext context = new DynamoDBContext(client);

            var table = Table.LoadTable(client, "user-profile");
            var item = await table.GetItemAsync(id);

            item["image"] = fileName;

            await table.PutItemAsync(item);
        }
    }
}
```

As parameter to our function we get event from S3, based on its information we obtain user id from file metadata `x-amz-meta-user-profile-id` - and update necessary item in DynamoDB table. To deploy we use the same command as before:

```
dotnet lambda deploy-function SetProfileImage `
    --function-role ProfileImageRole `
    --region eu-west-1
```

There one more thing before we can test our function, we need to give S3 permission to trigger our lambda. In AWS there are two event models:
- push - all non-streaming AWS services like S3 - in this case - event originator (S3) need to have permission to trigger lambda function.
- pull - this is for stream based services - DynamoDB and Kinessis - lambda need to have access to event originator service.

Those two model also differ in approach to the error handling. In case of unhandled exception push will retry two times and pull is going to retry until stream data expires which can take some time.

In case of this function we are using push model, and we need to allow S3 to trigger our lambda:

```
aws lambda add-permission `
--function-name SetProfileImage `
--action lambda:InvokeFunction `
--principal s3.amazonaws.com `
--source-arn arn:aws:s3:::user-profile-images-01234 `
--statement-id 1
```

And create trigger itself:

```
aws s3api put-bucket-notification-configuration `
--bucket user-profile-images-01234 `
--notification-configuration file://notification.json
```

Notification configuration looks as below, notice that function will be triggered only when image is uploaded into `original` folder with `jpg` extension. Thumbnail image we will be uploading into `thumbnails` folder to avoid setting this trigger again for already processes image - the 'forever' loop I described earlier.

``` json
{
"LambdaFunctionConfigurations": [
    {
      "Id": "my-lambda-function-s3-event-configuration",
      "LambdaFunctionArn": "arn:aws:lambda:eu-west-1:980358546491:function:SetProfileImage",
      "Events": [ "s3:ObjectCreated:Put" ],
      "Filter": {
        "Key": {
          "FilterRules": [
            {
              "Name": "prefix",
              "Value": "original/"
            },             
            {
              "Name": "suffix",
              "Value": ".jpg"
            }]
        }
      }
    }]
}
```

Now we can test this part of scenario - by uploading image to S3 with necessary metadata:

```
aws s3 cp .\maverick.jpg s3://user-profile-images-01234/original/maverick.jpg `
    --metadata '{\"user-profile-id\":\"1\"}'
```

`User-profile-id` key will be prepended with `x-amz-meta` by AWS - more about this can be found [here][4].

Let's verify by checking bucket for image and query table for the changes:

```
aws s3 ls s3://user-profile-images-01234/original/

aws dynamodb scan --table-name user-profile
```

We can also inspect our function execution logs using:

```
aws logs get-log-events `
    --log-group-name '/aws/lambda/SetProfileImage' `
    --log-stream-name '2018/05/02/[$LATEST]1b8983ef5fa84777a9217d1a77b41fbf'
```

So the first part is done - we can create our second function and its role.

```
dotnet new lambda.EmptyFunction `
    --name CreateThumbnail `
    --profile default `
    --region eu-west-1

aws iam create-role `
    --role-name CreateThumbnailRole `
    --assume-role-policy-document file://role.json

aws iam put-role-policy `
    --role-name CreateThumbnailRole `
    --policy-name CreateThumbnailRoleInlinePolicy `
    --policy-document file://create_thumbnail_policy.json        
```

Role policy, one again I'm not restricting it to specific resource:

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetRecords",
                "dynamodb:GetShardIterator",
                "dynamodb:DescribeStream",
                "dynamodb:ListStreams",
                "logs:CreateLogStream",
                "logs:CreateLogGroup",
                "logs:PutLogEvents",
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:PutObjectTagging"
            ],
            "Resource": "*"
        }
    ]
}
```

This function will require following nuget packages:

```
dotnet add package Amazon.Lambda.DynamoDBEvents
dotnet add package Amazon.Lambda.S3Events
dotnet add package TinyPNG
```

And its code should look something like this:

``` csharp
namespace CreateThumbnail
{
    public class Function
    {
        private string _metadataKey = "x-amz-meta-user-profile-id";
        private readonly string _imageType = ".jpg";
        private readonly AmazonS3Client _s3Client;
        private static readonly JsonSerializer _jsonSerializer = new JsonSerializer();

        public Function()
        {
            _s3Client = new AmazonS3Client();
        }

        public async Task FunctionHandler(DynamoDBEvent dynamoEvent, ILambdaContext context)
        {
            foreach (var record in dynamoEvent.Records)
            {
                if(!record.Dynamodb.NewImage.ContainsKey("image") || !record.Dynamodb.Keys.ContainsKey("id"))
                {
                    context.Logger.Log("Missing data.");
                    continue;
                }

                string id = record.Dynamodb.Keys["id"].N;
                string filePath = record.Dynamodb.NewImage["image"].S;
                string fileName = Path.GetFileName(filePath);
                
                if (_imageType != Path.GetExtension(fileName).ToLower())
                {
                    context.Logger.Log($"Not a supported image type");
                    continue;
                }

                try
                {
                    string bucketName = Environment.GetEnvironmentVariable("BucketName");
                    var tinyPngKey = Environment.GetEnvironmentVariable("TinyPngKey");

                    using (var objectResponse = await _s3Client.GetObjectAsync(bucketName + "/original", fileName))
                    using (Stream responseStream = objectResponse.ResponseStream)
                    {
                        TinyPngClient tinyPngClient = new TinyPngClient(tinyPngKey);

                        using (var downloadResponse = await tinyPngClient
                            .Compress(responseStream)
                            .Resize(150, 150)
                            .GetImageStreamData())
                        {
                            var putRequest = new PutObjectRequest
                            {
                                BucketName = bucketName + "/thumbnails",
                                Key = fileName,
                                InputStream = downloadResponse,
                                TagSet = new List<Tag>
                                {
                                    new Tag
                                    {
                                        Key = "Thumbnail", Value = "true"
                                    },
                                },
                            };

                            putRequest.Metadata.Add(_metadataKey, id);

                            await _s3Client.PutObjectAsync(putRequest);
                        }
                    }
                }
                catch (Exception ex)
                {
                    context.Logger.Log($"Exception: {ex}");
                }
            }
        }
    }
}
```

Function will be triggered by event(s) from DynamoDB - after we verify all the checks we obtain file from S3, create thumbnail image ([TinyPNG][1]) and store it back to S3. We are using here environment variables, so let's define them, and deploy the function:

```
aws lambda update-function-configuration `
    --function-name CreateThumbnail `
    --region eu-west-1 `
    --environment "Variables={TinyPngKey=XYZ123,BucketName=user-profile-images-01234}"

dotnet lambda deploy-function CreateThumbnail `
    --function-role CreateThumbnailRole `
    --region eu-west-1
```

Now we need to create event mapping - telling lambda to pull data from DynamoDB stream. 

```
aws lambda create-event-source-mapping `
    --region eu-west-1 `
    --function-name CreateThumbnail `
    --event-source arn:aws:dynamodb:eu-west-1:111111111111:table/user-profile/stream/2018-05-02T20:53:17.155 `
    --batch-size 10 `
    --starting-position TRIM_HORIZON
```

To get the necessary ARN of DynamoDB Steam we can read `LatestStreamArn` property from:

```
aws dynamodb describe-table --table-name user-profile
```

And voilà - we are done. To test whole scenario let's upload image:

```
aws s3 cp .\iceman.jpg s3://user-profile-images-01234/oroginal/iceman.jpg `
    --metadata '{\"user-profile-id\":\"2\"}'
```

We should see new file created in thumbnail folder:

```
aws s3 ls s3://user-profile-images-01234/thumbinals
```

[1]: https://tinypng.com
[2]: https://tinypng.com/developers
[3]: https://github.com/sakowiczm/aws-profile-image
[4]: https://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectPUT.html

