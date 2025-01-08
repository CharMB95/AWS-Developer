# Lab 1  Create an Amazon DynamoDB Table and Add Data

1. Create a DynamoDB table using the AWS Management Console or AWS CLI
- Table name: `BootcampUsers`
- Primary partition key: `UserId` (String)
- Primary sort key: `Timestamp` (Number)
- Local secondary index name: `LastNameIndex`, partition key: `UserID`, sort key: `LastName`,  projected attributes: `All`

CLI command:

```
aws dynamodb create-table --table-name BootcampUsers --attribute-definitions AttributeName=UserId,AttributeType=S AttributeName=Timestamp,AttributeType=N AttributeName=LastName,AttributeType=S --key-schema AttributeName=UserId,KeyType=HASH AttributeName=Timestamp,KeyType=RANGE --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 --local-secondary-indexes 'IndexName=LastNameIndex,KeySchema=[{AttributeName=UserId,KeyType=HASH},{AttributeName=LastName,KeyType=RANGE}],Projection={ProjectionType=ALL}'
```
2. Use the AWS CLI to put the items into the DynamoDB table

```
aws dynamodb batch-write-item --request-items file://ddb-users-table.json
```

# Lab 2 - Querying Data with Partition Keys and Indexes

1. Create a GSI (Global Secondary Index) using the AWS Management Console or AWS CLI:
- Index name: `BootcampUsersByLastName`
- Partition key: `LastName` (String)
- Sort key: `Timestamp` (Number)

CLI command:

```
aws dynamodb update-table --table-name BootcampUsers --attribute-definitions AttributeName=LastName,AttributeType=S AttributeName=Timestamp,AttributeType=N --global-secondary-index-updates "Create={IndexName=BootcampUsersByLastName,KeySchema=[{AttributeName=LastName,KeyType=HASH},{AttributeName=Timestamp,KeyType=RANGE}],Projection={ProjectionType=ALL},ProvisionedThroughput={ReadCapacityUnits=5,WriteCapacityUnits=5}}"
```

2. Perform queries and scans using partition keys and secondary indexes:

- Scan the whole table

aws dynamodb scan --table-name BootcampUsers

- Query the LSI (LastNameIndex) using the partition key (UserId) and the sort key (LastName) of the index

```
aws dynamodb query --table-name BootcampUsers --index-name LastNameIndex --key-condition-expression "UserId = :userId AND LastName = :lastName" --expression-attribute-values '{":userId": {"S": "U1"},":lastName": {"S": "Smith"}}'
```

This command queries the BootcampUsers table using the LastNameIndex LSI. You would use this command when you want to fetch data for a specific user with a particular last name within the same partition.

- Query the GSI (BootcampUsersByLastName) using the partition key (LastName) and sort key (Timestamp)

```
aws dynamodb query --table-name BootcampUsers --index-name BootcampUsersByLastName --key-condition-expression "#ln = :lastName AND #ts = :timestamp" --expression-attribute-names '{"#ln": "LastName","#ts": "Timestamp"}' --expression-attribute-values '{":lastName": {"S": "Smith"},":timestamp": {"N": "1631083945"}}'
```

This command queries the BootcampUsers table using the BootcampUsersByLastName GSI. You would use this command when you want to fetch data based on last name and timestamp, which are not the primary keys in the base table.


# Lab 3 - Creating and Using AWS Lambda Functions with Amazon DynamoDB

1. Create an AWS Lambda function that reads data from the DynamoDB table:

- Sign in to the AWS Management Console and open the AWS Lambda console
- Click "Create function" and choose "Author from scratch"
- Enter the following details for your Lambda function:
- Function name: "ReadDynamoDB"
- Runtime: "Node.js 16.x"
- Under "Function code" select "Edit code inline" and paste the following code (Node.js)

```javascript
const AWS = require("aws-sdk");
const dynamodb = new AWS.DynamoDB.DocumentClient();

exports.handler = async (event) => {
    const params = {
        TableName: "BootcampUsers",
    };

    try {
        const data = await dynamodb.scan(params).promise();
        return data.Items;
    } catch (error) {
        console.error(error);
        throw error;
    }
};
```

2. Attach the `AmazonDynamoDBReadOnlyAccess` IAM policy to the function execution role
3. Invoke the Lambda function and view the results

- In the AWS Lambda console, select your "ReadDynamoDB" function
- Click "Test" in the top right corner
- In the "Configure test event" window, choose "Create new test event" and enter a name for the event (e.g., "TestReadDynamoDB"). You can leave the event JSON as-is
- Click "Save" and then click "Test" to invoke your Lambda function
- Check the "Execution results" and "Function logs" to verify that the function executed correctly and returned the items from the DynamoDB table

# Lab 4 - Working with Amazon DynamoDB Streams and Lambda Functions

1. Enable a DynamoDB stream on the BootcampUsers table

- In the AWS Management Console, navigate to the Amazon DynamoDB console
- Click "Tables" > "Update settings' in the left sidebar and select the "BootcampUsers" table
- Click the "Turn on" tab for DynamoDB streams
- Choose "New and old images" for the "view type" and click "Turn on stream"

2. Create a Lambda function to process the stream events

- In the AWS Lambda console, click "Create function" and choose "Author from scratch"
- Enter the following details for your Lambda function:
- Function name: "ProcessDynamoDBStream"
- Runtime: "Node.js 16.x"
- Under "Function code," select "Edit code inline" and paste the following code (Node.js)

```javascript
exports.handler = async (event) => {
    console.log("Processing DynamoDB Stream Event:", JSON.stringify(event, null, 2));

    for (const record of event.Records) {
        console.log("DynamoDB record:", JSON.stringify(record.dynamodb, null, 2));
    }
};
```

- Add the necessary permissions to DynamoDB using the following policy data

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "APIAccessForDynamoDBStreams",
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetRecords",
                "dynamodb:GetShardIterator",
                "dynamodb:DescribeStream",
                "dynamodb:ListStreams"
            ],
            "Resource": "*"
        }
    ]
}
```

3. Configure the Lambda function to be triggered by the DynamoDB stream

- In the AWS Lambda console, select your "ProcessDynamoDBStream" function
- Scroll down to the "Function overview" section and click "Add trigger"
- Choose "DynamoDB" as the trigger type
- Select the "BootcampUsers" table from the "DynamoDB table" dropdown menu
- For "Batch size," enter "100"
- Enable the "Enable trigger" checkbox and click "Add" to add the trigger

4. Test the solution by running some commands to add data to the table

```
aws dynamodb put-item --table-name BootcampUsers --item '{"UserId": {"S": "U26"},"Timestamp": {"N": "1631083970"},"FirstName": {"S": "Zach"},"LastName": {"S": "Walker"},"Age": {"N": "31"}}'
```

- Check CloudWatch Logs for an entry showing that the record was processed

# Lab 5 - Implementing DynamoDB Global Tables

1. Create two DynamoDB tables in different AWS Regions

- Follow the instructions from Lab 1 to create a DynamoDB table named "BootcampUsers" in two different AWS Regions (e.g., us-east-1 and us-west-2)
- After creating both tables, enable "Global Table" on the "Global tables" tab in the DynamoDB console for each table

2. Add data to one table and observe the replication to the other table

- Use the AWS CLI or AWS Management Console to add items to one of the tables

- Verify that the data is replicated to the other table by checking the contents of the second table using the AWS Management Console or the AWS CLI

# Lab 6 - Using Time to Live (TTL) with Amazon DynamoDB

1. Enable TTL on the `BootcampUsers` table

- With the table selected, click "Actions" and "Turn on TTL"
- Enter "ExpirationTime" in the "TTL attribute" field and click "Turn on TTL"

2. Add a TTL attribute to existing items

- Calculate the desired expiration time as a Unix timestamp
- Update the items with a new attribute `ExpirationTime` containing the calculated Unix timestamp
- Use the AWS CLI to update the items in the DynamoDB table with the new attribute

```
aws dynamodb update-item --table-name BootcampUsers --key '{ "UserId": {"S": "U1"}, "Timestamp": {"N": "1631083945"}}' --update-expression "SET ExpirationTime = :expirationTime" --expression-attribute-values '{":expirationTime": {"N": "<YOUR_EXPIRATION_TIME>"}}'
```

Replace `<YOUR_EXPIRATION_TIME>` with the calculated Unix timestamp. Note that DynamoDB can take up to 48 hours to delete the item after expiration

# Lab 7 - Backup, Restore, and Export Data from DynamoDB

1. Create an On-Demand Backup of the DynamoDB Table:
   - Go to the AWS Management Console, navigate to the DynamoDB service
   - Select the table you want to back up
   - Choose the "Backups" tab, and click on "Create backup"

2. Restore Data from a Backup:
   - Delete an item from the DynamoDB table using the AWS CLI

```
aws dynamodb delete-item --table-name BootcampUsers --key '{"UserId": {"S": "U2"}, "Timestamp": {"N": "1631083946"}}'
```
   - Locate the backup you created in step 1, click on the backup name
   - Click "Restore backup" and provide the details for the new table (e.g., "BootcampUsersRestored")
   - Wait for the restore process to complete and verify the restored data in the new table

3. Schedule Automated Backups (Point-In-Time Recovery):
   - Go to the AWS Management Console, navigate to the DynamoDB service
   - Select the table for which you want to enable PITR
   - In the "Overview" tab, locate the "Point-in-time recovery" section
   - Click "Enable" and confirm

   - Describe the backup configuration using the AWS CLI

```
aws dynamodb describe-continuous-backups --table-name BootcampUsers
```

4. Export Data from DynamoDB to Amazon S3
   - Create an Amazon S3 bucket using the AWS CLI

```
aws s3api create-bucket --bucket my-dynamodb-export-bucket --region us-east-1
```

   - Export the DynamoDB table data to the S3 bucket using the AWS Management Console:
     - Go to the DynamoDB service
     - Select the table you want to export
     - Choose the "Exports and streams" tab
     - Click on "Export to S3"
     - Select the Amazon S3 bucket you created, provide an export name, and click "Export"

   - Verify the exported data in the S3 bucket using the AWS CLI

```
aws s3 ls s3://my-dynamodb-export-bucket/
```

- Find the exported data, download, and review the information