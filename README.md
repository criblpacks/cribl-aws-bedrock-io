# AWS Bedrock Invocation Logs Pack
----

## About this Pack

This pack is built as a complete SOURCE + DESTINATION solution (identified by the IO suffix). Data collection and delivery happen entirely within the Pack's context - you can choose how data arrives at a DESTINATION:
*  *Send to Worker Group Routes* (the default): data is sent to the top-level Worker Group Routes.
*  *Default Destination*: data is sent to the Worker Group's [Default Destination](https://docs.cribl.io/stream/destinations-default/). 
*  *In-Pack Destination*: data is sent to one or more Destinations configured within the Pack.

This Pack enables comprehensive ingestion, processing, and routing of AWS Bedrock telemetry into Cribl Stream for normalization, enrichment, and delivery to downstream destinations such as Amazon S3 or Splunk.
It captures three complementary data sources for full visibility into Bedrock usage and security:

* `Invocation Logs`: Actual prompts, responses, token counts, latency.
* `Management Events`: Captures admin actions such as creating agents, modifying guardrails, changing configs.
* `Data Events`: Data plane operations like agent invocations, async jobs, flow executions.

This Pack provides the following benefits:

* Ingests all three Bedrock log types from Amazon S3
* Uses IAM AssumeRole for secure, short-lived access from Cribl Cloud
* Supports event-driven SQS-based collection
* Pre-configured routes and pipelines for each log type
* Normalizes events into clean, structured JSON with consistent field naming
* Filters CloudTrail Management events to Bedrock-only in the pipeline
* Supports routing to Amazon S3, Splunk, or other Cribl-supported destinations

## Deployment

* Every bundled Source within this pack adds a hidden field: `__packsource`. This field allows for simplified routing based on the Pack source.
* This pack is configured by default to use the Destination *Send to Worker Group Routes*. You *must* add either a Worker Group Route or rely on the Default Destination.
* To explicitly use the Worker Group's *Default Destination*, change the Pack's Routes to *default:default*. The Pack will then route the data to the destination currently set as the Default on the Worker Group.

This section describes all required steps to deploy this Pack, including AWS-side setup and Cribl configuration.

### Prerequisites

* An AWS account with Amazon Bedrock enabled
* Permissions to create IAM roles and attach policies
* Permissions to configure S3 buckets and event notifications
* Permissions to create SQS queues
* Permissions to configure CloudTrail trails

### Part 1: Bedrock Invocation Logs
Invocation logs capture the actual model interactions — prompts, responses, token counts, and latency metrics.

### `Step 1.1: Configure the S3 Bucket (AWS)`

* Identify or create an S3 bucket to store Bedrock invocation logs. Example bucket name: <YOUR_COMPANY>-bedrock-invocation-logs
* Ensure Bedrock logs are written under the prefix AWSLogs/<AWS_ACCOUNT_ID>/BedrockModelInvocationLogs/
* Verify that JSON or JSON.GZ files are present in the bucket

### `Step 1.2: Enable Bedrock Model Invocation Logging (AWS)`

* Navigate to the AWS Console → Amazon Bedrock
* Go to Settings → Model invocation logging
* Enable logging and select Amazon S3 as the destination
* Choose the S3 bucket created in Step 1.1
* Logs will be delivered to: AWSLogs/<AWS_ACCOUNT_ID>/BedrockModelInvocationLogs/

Verify logging is working:
* Open the Bedrock Playground and submit a test prompt
* Check the S3 bucket for .json or .json.gz files (allow 5-10 minutes)

### `Step 1.3: Create the SQS Queue for Invocation Logs (AWS)`
* Create an SQS queue to receive S3 event notifications:
```
aws sqs create-queue \
  --queue-name <YOUR_COMPANY>-bedrock-invocation-notifications \
  --region <YOUR_REGION>
```

### `Step 1.4: Configure the SQS Queue Policy (AWS)`
* Add a policy to allow S3 to send notifications to the queue:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:<REGION>:<AWS_ACCOUNT_ID>:<YOUR_COMPANY>-bedrock-invocation-notifications",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::<YOUR_COMPANY>-bedrock-invocation-logs"
        }
      }
    }
  ]
}
```
### `Step 1.5: Configure S3 Event Notifications (AWS)`

* Navigate to S3 → Your invocation logs bucket → Properties
* Scroll to Event notifications → Create event notification
* Configure:
   * Event name: invocation-logs-created
   * Event types: s3:ObjectCreated:*
   * Destination: SQS queue → Select your invocation notifications queue
* Save

### Part 2: CloudTrail Logs (Management + Data Events)

CloudTrail captures API-level activity for security auditing and compliance.

### `Step 2.1: Create the S3 Bucket for CloudTrail Logs (AWS)`

* Create or identify an S3 bucket to store CloudTrail logs.
   * Example bucket name: <YOUR_COMPANY>-bedrock-cloudtrail-logs

### `Step 2.2: Create a CloudTrail Trail (AWS)`

* Navigate to AWS Console → CloudTrail → Create trail
* Trail name: <YOUR_COMPANY>-bedrock-cloudtrail
* Storage location: Select the S3 bucket from Step 2.1
* Management events: Enable (Read + Write)
* Data events: Skip for now (configured via CLI in the next step)
* Create the trail

### `Step 2.3: Configure CloudTrail Data Event Selectors (AWS)`

* CloudTrail Data events require explicit resource type selectors. Run the following command to enable Bedrock data events:
```
aws cloudtrail put-event-selectors \
  --trail-name <YOUR_COMPANY>-bedrock-cloudtrail \
  --advanced-event-selectors '[
    {"Name": "Bedrock AgentAlias", "FieldSelectors": [{"Field": "eventCategory", "Equals": ["Data"]}, {"Field": "resources.type", "Equals": ["AWS::Bedrock::AgentAlias"]}]},
    {"Name": "Bedrock AsyncInvoke", "FieldSelectors": [{"Field": "eventCategory", "Equals": ["Data"]}, {"Field": "resources.type", "Equals": ["AWS::Bedrock::AsyncInvoke"]}]},
    {"Name": "Bedrock AutomatedReasoningPolicy", "FieldSelectors": [{"Field": "eventCategory", "Equals": ["Data"]}, {"Field": "resources.type", "Equals": ["AWS::Bedrock::AutomatedReasoningPolicy"]}]},
    {"Name": "Bedrock Blueprint", "FieldSelectors": [{"Field": "eventCategory", "Equals": ["Data"]}, {"Field": "resources.type", "Equals": ["AWS::Bedrock::Blueprint"]}]},
    {"Name": "Bedrock DataAutomationInvocation", "FieldSelectors": [{"Field": "eventCategory", "Equals": ["Data"]}, {"Field": "resources.type", "Equals": ["AWS::Bedrock::DataAutomationInvocation"]}]},
    {"Name": "Bedrock FlowAlias", "FieldSelectors": [{"Field": "eventCategory", "Equals": ["Data"]}, {"Field": "resources.type", "Equals": ["AWS::Bedrock::FlowAlias"]}]}
  ]'
  ```
`Note:` CloudTrail limits *resources.type* to one value per selector, so each Bedrock resource type requires its own selector entry.

### `Step 2.4: Create the SQS Queue for CloudTrail Logs (AWS)`
```
bashaws sqs create-queue \
  --queue-name <YOUR_COMPANY>-bedrock-cloudtrail-notifications \
  --region <YOUR_REGION>
Step 2.5: Configure the SQS Queue Policy (AWS)
json{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:<REGION>:<AWS_ACCOUNT_ID>:<YOUR_COMPANY>-bedrock-cloudtrail-notifications",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::<YOUR_COMPANY>-bedrock-cloudtrail-logs"
        }
      }
    }
  ]
}
```

### `Step 2.6: Configure S3 Event Notifications (AWS)`

* Navigate to S3 → Your CloudTrail logs bucket → Properties
* Scroll to Event notifications → Create event notification
* Configure:
   * Event name: cloudtrail-logs-created
   * Event types: s3:ObjectCreated:*
   * Destination: SQS queue → Select your CloudTrail notifications queue
* Save

### Part 3: IAM Role Configuration
Create a single IAM role that Cribl will assume for accessing both S3 buckets and SQS queues.

`### Step 3.1: Create the IAM Role (AWS)`

* Navigate to AWS Console → IAM → Roles → Create role
* Trusted entity type: AWS account
* Select Another AWS account
* Enter the Cribl Cloud AWS Account ID (found in your Cribl Cloud worker settings)
* Role name: CriblBedrockLogsReader

`### Step 3.2: Attach the IAM Policy (AWS)`

* Create and attach a policy with the following permissions:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3BucketList",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": [
        "arn:aws:s3:::<YOUR_COMPANY>-bedrock-invocation-logs",
        "arn:aws:s3:::<YOUR_COMPANY>-bedrock-cloudtrail-logs"
      ]
    },
    {
      "Sid": "S3ObjectRead",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": [
        "arn:aws:s3:::<YOUR_COMPANY>-bedrock-invocation-logs/AWSLogs/*",
        "arn:aws:s3:::<YOUR_COMPANY>-bedrock-cloudtrail-logs/AWSLogs/*"
      ]
    },
    {
      "Sid": "SQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl"
      ],
      "Resource": [
        "arn:aws:sqs:<REGION>:<AWS_ACCOUNT_ID>:<YOUR_COMPANY>-bedrock-invocation-notifications",
        "arn:aws:sqs:<REGION>:<AWS_ACCOUNT_ID>:<YOUR_COMPANY>-bedrock-cloudtrail-notifications"
      ]
    }
  ]
}
```

* Copy the Role ARN after creation.

## Configure Output Format

Each data type can be configured to output data in either normalized JSON or Splunk (`_raw` + Splunk fields) format. Enable *only one* format for each pipeline.

## Configure your Destination/Update Pack Routes
To ensure proper data routing, you must make a choice: retain the current setting to use the Default Destination defined by your Worker Group, or define a new Destination directly inside this pack and adjust the pack's route accordingly.

## Enable the required inputs

### Commit and Deploy
Once everything is configured, perform a Commit & Deploy to enable data collection.

## Pack Configurable Items 
The following are the in-Pack configurable items - review/update them as needed. 

The Pack has the following variables:

* `default_splunk_index`: Default index for the Splunk output - defaults to `aws`. 
* `default_splunk_sourcetype`: Default sourcetype for the Splunk output - defaults to `aws:bedrock`. 
* `aws_region`: AWS region where your S3 buckets and SQS queues are located - defaults to us-east-1. 
* `iam_role_arn`: IAM Role ARN that Cribl will assume for cross-account access to S3 and SQS.
* `bedrockl_cloudtrail_sqs_queue`: SQS queue URL for CloudTrail log notifications. 
* `bedrock_invocation_sqs_queue`: SQS queue URL for Bedrock invocation log notifications.

## References

- [S3 Collector Doc](https://docs.cribl.io/stream/collectors-s3/)
- [Monitor Model Invocation](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html)

## Upgrades

Upgrading certain Cribl Packs using the same Pack ID can have unintended consequences. See [Upgrading an Existing Pack](https://docs.cribl.io/stream/packs#upgrading) for details.

## Release Notes
### Version 1.0.1
* Updated Route Destinations to "Send to Worker Group Routes". See above for details.

### Version 1.0.0
Initial release

## Contributing to the Pack

To contribute to the Pack, please connect with us on [Cribl Community Slack](https://cribl-community.slack.com/). You can suggest new features or offer to collaborate.

## License
This Pack uses the following license: [Apache 2.0](https://github.com/criblio/appscope/blob/master/LICENSE).





