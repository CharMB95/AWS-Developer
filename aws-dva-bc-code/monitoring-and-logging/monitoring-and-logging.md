# Exercise 1: Setting up AWS Key Management Service (KMS)

1. From the AWS Management Console, go to the "Security, Identity, & Compliance" section and click on "KMS" (Key Management Service).
2. Click on "Create a key" in the center of the page.
3. Choose "Symmetric" for Key type and click "Next".
4. Provide an alias for the key, e.g., "MyCloudTrailKey", leave key material origin as "KMS", and click "Next".
5. Skip adding tags (unless needed for your organization's resource tracking), click "Next".
6. Define the key administrative permissions by selecting IAM users or roles that can administer the key, then click "Next".
7. Define the key usage permissions by selecting IAM users or roles that can use the key to encrypt and decrypt data, and add permissions for CloudTrail by editing the final JSON policy and adding the following code:

```json
        {
            "Sid": "Allow CloudTrail to use the key",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": [
                "kms:GenerateDataKey*",
                "kms:Encrypt"
            ],
            "Resource": "*"
        }
```

# Exercise 2: Setting up Logging with CloudTrail

1. In the AWS Management Console, go to the "Management & Governance" section and click on "CloudTrail".
2. Click on "Create trail". 
3. Fill in the Trail name, e.g., "MyTrail". 
4. In the "Storage location" section, choose "New S3 bucket" and provide a unique Bucket name.
5. Under "Advanced settings", set "Log file SSE-KMS encryption" to enabled and select the key alias you just created in the KMS section, then click "Create".
6. Create a CloudWatch log group with a new group name, allow the creation of a new role, and capture management events.

# Exercise 3: Metric Filters, Subscriptions, and Alarms

## Create a metric filter

1. In the AWS Management Console, go to the "Management & Governance" section and click on "CloudWatch".
2. On the CloudWatch home page, click on "Log groups" on the left panel.
3. Click on the Log group you created in the previous step
4. Click "Create Metric Filter" button.
5. In the filter pattern field, enter the code

```json
 { ($.eventName = DeleteBucket) }
 ```
 
 This filter pattern matches AWS CloudTrail log events in which the `eventName` field is "DeleteBucket".
 
6. Enter the values as per below:

- Filter Name: This is an optional field. You can put something like "DeleteBucketFilter" here.
- Metric Namespace: This is a container for your metrics. You could put something like "CloudTrailMetrics" here.
- Metric Name: This is the name that identifies the metric. You could put "DeleteBucketCount" here.
- Metric Value: This is the value to publish to the metric when the pattern set in the filter pattern is found in a log event. Enter "1".
- Default Value: The default value is published to the metric when the pattern does not match. If you leave this blank, no value is published when there is no match. Do not enter a value.
- Unit: This is the unit of measurement used for the metric. Since we're counting the occurrences of a specific event, you can choose "Count" as the unit.
- Dimensions: These are name-value pairs that help you to filter a metric. You can leave this empty for this case.

7. Click "Create metric filter".

Once the metric filter is created, it will start to count the `DeleteBucket` events and store them as data points in the `DeleteBucketCount` metric. You can then set alarms on this metric to get notified whenever `DeleteBucket` operations are detected.

Run a test by deleting a bucket and then running a metric filter test pattern against the log stream.

## Create an Alarm on the Metric:

1. On the CloudWatch home page, click on "Alarms" on the left panel.
2. Click on "Create Alarm".
3. In the "Create Alarm" wizard, click on "Select metric".
4. In the "All metrics" tab, find the "CloudTrailMetrics" namespace, and then click on the "DeleteBucketCount" metric.
5. Click "Select metric".
6. In the Conditions section, set the Threshold type to "Static". For "Whenever DeleteBucketCount is...", select "Greater/Equal" in the first box and enter "1" in the second box.
7. Click "Next" and configure actions as per your preferences (like sending notifications).
8. Click "Next", add alarm name and description, and finally click "Create Alarm".

With these steps, we've set up a filter to track `DeleteBucket` operations, and an alarm that notifies us when such operations occur.

# Exercise 4: Create a subscription filter for logging bucket creations to DynamoDB

## Create a DynamoDB table

1. Open the AWS Management Console and navigate to the DynamoDB service.
2. Click on "Create table".
3. Fill in the "Table name" (e.g., BucketInfo) and the "Primary key" (e.g., BucketName). Leave other settings as default.
4. Click on "Create".

## Create a Lambda function

1. Navigate to the Lambda service in the AWS Console.
2. Click on "Create function".
3. Fill in the "Function name" (e.g., ProcessLogData) and choose "Python 3.x" as the "Runtime".
4. Under "Permissions", choose "Create a new role with basic Lambda permissions".
5. Click on "Create function".
6. In the function code editor, input the following sample Python code:

```python
import json
import gzip
import base64
import boto3

def lambda_handler(event, context):
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('BucketInfo')

    data = event['awslogs']['data']
    decoded = json.loads(gzip.decompress(base64.b64decode(data)))
    log_events = decoded['logEvents']

    for log_event in log_events:
        detail = json.loads(log_event['message'])
        if detail['eventName'] == 'CreateBucket':
            response = table.put_item(
                Item={
                    'BucketName': detail['requestParameters']['bucketName'],
                    'Creator': detail['userIdentity']['arn'],
                    'Time': detail['eventTime']
                }
            )
```
7. Assign permissions to read and write to DynamoDB

This function processes the log data from CloudWatch Logs, finds CreateBucket events, and writes relevant information to the BucketInfo DynamoDB table.

## Create the subscription filter

1. Navigate back to the CloudWatch service in the AWS Console.
2. In the left panel, under "Logs", click on "Log groups".
3. Click on the Log group that you have created earlier, e.g., "MyLogGroup".
4. At the top of the "MyLogGroup" page, click on "Create Lambda subscription Filter".
5. In the "Create Lambda subscription Filter" wizard, you'll be asked to define a pattern. In the "Filter pattern" box, enter the code:

```json
{ ($.eventName = CreateBucket) }
 ```

6. Choose the Lambda function ProcessLogData as the destination.
7. Click on "Start Streaming".
8. Test by creating an S3 bucket and within a few minutes information about the event should be added to the DynamoDB table


# Exercise 5: Query log data with Athena

1. Open the CloudTrail console.
2. In the navigation pane, choose Event history.
3. Choose Create Athena table.
4. For Storage location, use the down arrow to select the Amazon S3 bucket where log files are stored for the trail to query.
5. Choose Create table. The table is created with a default name that includes the name of the Amazon S3 bucket.
6. Navigate to Athena.
7. In the query editor, with the AwsDataCatalog data source and default database selected, run queries and view the results:

Create bucket events:
```sql
SELECT *
FROM "<table-name>"
WHERE eventname = 'CreateBucket';
```

Top 5 users by event count:
```sql
SELECT useridentity, COUNT(*) as event_count
FROM "<table-name>"
GROUP BY useridentity
ORDER BY event_count DESC
LIMIT 5;
```

Events by source IP address:
```sql
SELECT sourceipaddress, COUNT(*) as event_count
FROM "<table-name>"
GROUP BY sourceipaddress
ORDER BY event_count DESC;
```

Event count by Region:
```sql
SELECT awsregion, COUNT(*) as event_count
FROM "<table-name>"
GROUP BY awsregion
ORDER BY event_count DESC;
```

