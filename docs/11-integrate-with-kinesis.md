# Integrate with Kinesis
This section will walk through integration between 2 AWS services - Kinesis and Lambda.  

### Kinesis
Kinesis is a large scale, realtime event stream processing service from AWS. Kinesis is fully managed by AWS and 
offers less infrastructure maintenance to users. Kinesis can handle large scale streaming data at low latency.
Various sources can publish events to Kinesis and multiple consumers can connect and process the events.

#### Integration Example
AWS Lambda can be one of the consumers to process the records in Kinesis. AWS Lambda service can poll the Kinesis 
stream for the records and invoke a particular lambda function for processing.

The lambda function for this integration example will log the event received from kinesis. The log can be viewed using 
AWS cloud watch. The `lambda-cli-role` we created will be used here also.

#### (1) Setup Kinesis Stream
In this section we will create a Kinesis Stream. This stream will be used to publish events, for lambda to consume. 

##### (1.1) Create Stream 
The kinesis stream will be created with single shard. A shard is a uniquely identified sequence of 
data records in a stream. For our tutorial purpose, one shard is adequate. More concepts on Kinesis can be learnt from
[Kinesis Data Streams Terminology and Concepts](https://docs.aws.amazon.com/streams/latest/dev/key-concepts.html)

```shell script
➜ export AWS_PROFILE=lambda-cli-user
➜ aws kinesis create-stream --stream-name lambda-cli-stream --shard-count 1 --profile "$AWS_PROFILE"
```
Verify the creation of kinesis stream, by listing all the streams.
```
➜ aws kinesis list-streams --profile "$AWS_PROFILE"  
```
> Output:
```json
{ "StreamNames": [ "lambda-cli-stream" ] }
```

##### (1.2) Kinesis Stream ARN
We will need the Kinesis stream ARN for associating it with Lambda. This can be obtained by describing `lambda-cli-stream`.

```shell script
➜ export STREAM_NAME="lambda-cli-stream"
➜ aws kinesis describe-stream --stream-name "$STREAM_NAME" --profile "$AWS_PROFILE"
```
> Output:
```json
{
    "StreamDescription": {
        "Shards": [
            {
                "ShardId": "shardId-000000000000",
                "HashKeyRange": {
                    "StartingHashKey": "0",
                    "EndingHashKey": "123456789012345678901234567890123456789"
                },
                "SequenceNumberRange": {
                    "StartingSequenceNumber": "12345678901234567890123456789012345678901234567890123456"
                }
            }
        ],
        "StreamARN": "arn:aws:kinesis:us-east-1:919191919191:stream/lambda-cli-stream",
        "StreamName": "lambda-cli-stream",
        "StreamStatus": "ACTIVE",
        "RetentionPeriodHours": 24,
        "EnhancedMonitoring": [
            {
                "ShardLevelMetrics": []
            }
        ],
        "EncryptionType": "NONE",
        "KeyId": null,
        "StreamCreationTimestamp": 123456789.0
    }
}
```
##### (1.3) Set Stream ARN, Name
```
➜ export STREAM_NAME="lambda-cli-stream"
➜ export STREAM_ARN="arn:aws:kinesis:us-east-1:919191919191:stream/lambda-cli-stream"
```

#### (2) Build and deploy the lambda, to log Kinesis events 
We will build a simple lambda that will process kinesis events and log them. 

##### (2.1) Sample Kinesis Record Event
Stream records read from Kinesis, by Lambda, will have the format mentioned below.
```json
{
  "Records": [
    {
      "kinesis": {
        "kinesisSchemaVersion": "1.0",
        "partitionKey": "",
        "sequenceNumber": "",
        "data": "",
        "approximateArrivalTimestamp": 
      },
      "eventSource": "aws:kinesis",
      "eventVersion": "1.0",
      "eventID": "shardId-000000000001:",
      "eventName": "aws:kinesis:record",
      "invokeIdentityArn": "arn:aws:iam::919191919191:role/lambda-cli-role",
      "awsRegion": "us-east-1",
      "eventSourceARN": "arn:aws:kinesis:us-east-1:123456789012:stream/lambda-cli-stream"
    }
  ]
}
```
##### (2.2) Create the lambda
*Note:*
> The lambda will look for the data in key `record.kinesis.data`. 
> AWS CLI will encode the data sent to Kinesis streams into base64. Hence, the lambda will decode it back

```
➜ mkdir kinesis-event-logger-lambda
➜ cd kinesis-event-logger-lambda
➜ echo "exports.handler =  async (event, context) => {
  event.Records.forEach(record => console.log(Buffer.from(record.kinesis.data, 'base64').toString('utf8')));
};" > kinesisEventLogger.js
```
##### (2.3) Bundle & Deploy the lambda

```shell script
➜  export LAMBDA_ROLE_ARN=arn:aws:iam::919191919191:role/lambda-cli-role
➜  export AWS_REGION=us-east-1
➜  zip -r /tmp/kinesisEventLogger.js.zip kinesisEventLogger.js
➜  aws lambda create-function \
       --region "$AWS_REGION" \
       --function-name kinesisEventLogger \
       --handler 'kinesisEventLogger.handler' \
       --runtime nodejs10.x \
       --role "$LAMBDA_ROLE_ARN" \
       --zip-file 'fileb:///tmp/kinesisEventLogger.js.zip' \
       --profile "$AWS_PROFILE"
``` 
> output
```json
{
    "FunctionName": "kinesisEventLogger",
    "FunctionArn": "arn:aws:lambda:us-east-1:919191919191:function:kinesisEventLogger",
    "Runtime": "nodejs10.x",
    "Role": "arn:aws:iam::919191919191:role/lambda-cli-role",
    "Handler": "kinesisEventLogger.handler",
    "CodeSize": 332,
    "Description": "",
    "Timeout": 3,
    "MemorySize": 128,
    "LastModified": "2019-01-01T00:00:00.000+0000",
    "CodeSha256": "aBcDEeFG1H2IjKlM3nOPQrS4Tuv5W6xYZaB+7CdEf8g=",
    "Version": "$LATEST",
    "TracingConfig": {
        "Mode": "PassThrough"
    },
    "RevisionId": "123b9999-a1bb-1234-9a3b-777r2220a111"
}
```

#### (3) Map Kinesis to Lambda using event-source-mapping

Lambda can be mapped to the Kinesis stream in batches of events. We will map it for single event batch. This means,
for every event published in the Kinesis stream `lambda-cli-stream`, the `kinesisEventLogger` lambda will be
executed.

```shell script
➜  aws lambda create-event-source-mapping \
    --function-name kinesisEventLogger \
    --event-source "$STREAM_ARN" \
    --batch-size 1 \
    --starting-position LATEST \
    --profile "$AWS_PROFILE"
```
> Output :

```json
{
    "UUID": "12345678-a1b2-3cde-4567-89fg012hi34j",
    "BatchSize": 1,
    "MaximumBatchingWindowInSeconds": 0,
    "EventSourceArn": "arn:aws:kinesis:us-east-1:919191919191:stream/lambda-cli-stream",
    "FunctionArn": "arn:aws:lambda:us-east-1:919191919191:function:kinesisEventLogger",
    "LastModified": 1234567890.000,
    "LastProcessingResult": "No records processed",
    "State": "Enabled",
    "StateTransitionReason": "User action"
}
```
> Note: If you see the status as `Creating`, you can list the event sources mapped to `kinesisEventLogger` lambda
using `aws lambda list-event-source-mappings` and ensure its status is `Enabled`.  

#### (4) Publish an event in Kinesis to be processed by Lambda

Lets publish an event in the `lambda-cli-stream`. The event will be processed by lambda and we can check it in 
CloudWatch logs.

##### (4.1) Publish event
We will publish the event with message (data) _"Hello, Lambda CLI World"_

```shell script
➜  aws kinesis put-record --stream-name "$STREAM_NAME" \
    --partition-key 1 \
    --data "Hello, Lambda CLI World" \
    --profile "$AWS_PROFILE"
```

> Output:
```json
{
    "ShardId": "shardId-000000000000",
    "SequenceNumber": "12345678901234567890123456789012345678901234567890123456"
}
```

##### (4.2) View Lambda Log for kinesisEventLogger
Get the log group name, log stream name for kinesisEventLogger.
The latest `LOG_STREAM_NAME` will have the execution details.
> Note: The LOG_STREAM_NAME has `$` symbol and needs to be escaped with backslash.


```shell script
➜  export LOG_GROUP_NAME="/aws/lambda/kinesisEventLogger"
➜  aws logs describe-log-streams --log-group-name "$LOG_GROUP_NAME" --profile "$AWS_PROFILE"
➜  export LOG_STREAM_NAME="2019/12/13/[\$LATEST]aBcDEeFG1H2IjKlM3nOPQrS4Tuv5W6xYZaB"
➜  aws logs get-log-events --log-group-name "$LOG_GROUP_NAME" --log-stream-name "$LOG_STREAM_NAME"
```

Above commands with proper values should display the message "Hello, Lambda CLI World" in CloudWatch logs.

```json
{
    "timestamp": 1234567890123,
    "message": "2019-12-13T00:00:00.000Z\11111111-aaaa-bbbb-cccc-1234567890\tINFO\tHello, Lambda CLI World\n",
    "ingestionTime": 1234567890123
}
```

#### (5) Teardown
Lets remove the Kinesis stream and Lambda `kinesisEventLogger`.

```shell script
➜ aws kinesis delete-stream \
    --stream-name lambda-cli-stream \
    --profile "$AWS_PROFILE"

➜ aws lambda delete-function \
    --function-name kinesisEventLogger \
    --profile "$AWS_PROFILE"
```

🏁 **Congrats !** You learnt a key integration between AWS Lambda and Kinesis, for event processing. 🏁

**Next**: [Integrate with DynamoDB](12-integrate-with-dynamodb.md) 
