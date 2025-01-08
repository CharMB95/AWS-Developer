# Lab 1 - Deploying a Lambda Function using the AWS CLI and S3

1. Write a simple Lambda function in a file named `lambda_function.py`

```python
def lambda_handler(event, context):
    print("Lambda function invoked")
    return "Hello from Lambda!"
```

2. Zip the `lambda_function.py` file using the following command

```bash
zip function.zip lambda_function.py
```

3. Create a trust policy document named `trust-policy.json` with the following content

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

3. Create a new IAM role for your Lambda function with the necessary permissions

```bash
aws iam create-role --role-name simpleLambdaRole --assume-role-policy-document file://trust-policy.json
```

4. Attach the AWSLambdaBasicExecutionRole policy to the newly created role

aws iam attach-role-policy --role-name simpleLambdaRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

5. Use the AWS CLI to create the Lambda function

```bash
aws lambda create-function --function-name simpleLambda --runtime python3.10 --role arn:aws:iam::<ACCOUNT_ID>:role/simpleLambdaRole --handler lambda_function.lambda_handler --zip-file fileb://function.zip
```

6. Test the Lambda function to ensure it's working as expected:

```bash
aws lambda invoke --function-name simpleLambda --payload '{ "message": "Test invocation" }' output.txt --cli-binary-format raw-in-base64-out
```

7. Check the contents of the output.txt file to see the result of the invocation

## To update the existing Lambda function with code from an S3 location, follow these steps

1. Edit the message in the code, zip it up again, and upload the zipped file to an S3 bucket

```bash
aws s3 cp function.zip s3://YOUR_S3_BUCKET_NAME/function.zip
```

2. Update the existing Lambda function using the AWS CLI, specifying the S3 bucket and key for the function code

```bash
aws lambda update-function-code --function-name simpleLambda --s3-bucket YOUR_S3_BUCKET_NAME --s3-key function.zip
```

# Lab 2 - Initializing the AWS SDK Inside and Outside the Lambda Handler Function

1. Add permissions for S3 read only access to the function role
2. Modify the Lambda function from Exercise 1 to initialize the AWS SDK inside the Lambda handler function

```python
import boto3

def lambda_handler(event, context):
    print("Lambda function invoked")

    # Initialize the AWS SDK inside the handler function
    s3 = boto3.client('s3')

    # Add a simple S3 list operation using the initialized SDK
    bucket_name = 'my-cloudformation-s3-bucket-3121s2'

    try:
        response = s3.list_objects(Bucket=bucket_name)
        print("S3 objects:", response['Contents'])
        return "Hello from Lambda!"
    except Exception as error:
        print("Error:", error)
        raise error
```

- Zip the updated `lambda_function.py` file:

```bash
zip function.zip lambda_function.py
```

- Update the Lambda function using the AWS CLI:

```bash
aws lambda update-function-code --function-name simpleLambda --zip-file fileb://function.zip
```

3. Create a test event and run it several times from the console. Take a note of the duration of the initial execution and subsequent executions

4. Modify the Lambda function to initialize the AWS SDK outside the Lambda handler function

```python
import boto3

# Initialize the AWS SDK outside the handler function
s3 = boto3.client('s3')

def lambda_handler(event, context):
    print("Lambda function invoked")

    # Add a simple S3 list operation using the initialized SDK
    bucket_name = 'my-cloudformation-s3-bucket-3121s2'

    try:
        response = s3.list_objects(Bucket=bucket_name)
        print("S3 objects:", response['Contents'])
        return "Hello from Lambda!"
    except Exception as error:
        print("Error:", error)
        raise error
```

5. Zip the code and update the function

6. Test the function from the console again and compare the performance to the previous version


# Lab 3 - Storing Environment Variables and Secrets

1. Create a secret in AWS Secrets Manager. Prepare a JSON file named `secret.json` with the secret content

```json
{
  "api_key": "your_api_key_here"
}
```

2. Create the secret using the AWS CLI

```bash
aws secretsmanager create-secret --name MySecretApiKey --secret-string file://secret.json
```

3. Create a new Python file app.py for the Lambda function with the following code

```python
import os
import boto3
import json

def lambda_handler(event, context):
    secret_name = os.environ['SECRET_NAME']
    region_name = os.environ['AWS_REGION']

    session = boto3.session.Session()
    client = session.client(service_name='secretsmanager', region_name=region_name)
    response = client.get_secret_value(SecretId=secret_name)

    secret_data = json.loads(response['SecretString'])
    api_key = secret_data['api_key']

    # Use the API key to make a request to the hypothetical API
    # (Replace with actual API request code)
    print(f"API key: {api_key}")

    return {
        'statusCode': 200,
        'body': json.dumps('Hello from Lambda!')
    }
```

4. Zip your Lambda function code

```bash
zip function.zip app.py
```

5. Create a JSON file trust-policy.json with the following trust policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

6. Create the IAM role using the AWS CLI

```bash
aws iam create-role --role-name LambdaSecretsManagerRole --assume-role-policy-document file://trust-policy.json
```

7. Attach the required policies to the IAM role

```bash
aws iam attach-role-policy --role-name LambdaSecretsManagerRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam attach-role-policy --role-name LambdaSecretsManagerRole --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite
```

8. Retrieve the ARN of the newly created role

```bash
aws iam get-role --role-name LambdaSecretsManagerRole
```

9. Create the Lambda function using the AWS CLI, providing the IAM role ARN and environment variables

```bash
aws lambda create-function --function-name demo-environment-variables --runtime python3.10 --role REPLACE_WITH_YOUR_ROLE_ARN --handler app.lambda_handler --zip-file fileb://function.zip --environment Variables="{SECRET_NAME=REPLACE_WITH_YOUR_SECRET_ARN}"
```

10. Test the function. When the Lambda function is invoked, it should fetch the API key from AWS Secrets Manager and print it

```bash
aws lambda invoke --function-name demo-environment-variables --payload '{ "message": "Test invocation" }' output.txt --cli-binary-format raw-in-base64-out
```

11. Go to CloudWatch Logs to see if the API key was successfully printed

# Lab 4 - Connect a Lambda Function to a VPC

1. Create at least two private subnets in different Availability Zones within the default VPC
- Create a security group allowing necessary inbound and outbound traffic rules
- Create a route table for the private subnets and a NAT gateway
- Add policy code to allow Lambda to create interfaces and S3 write access

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
            "ec2:DescribeNetworkInterfaces",
            "ec2:CreateNetworkInterface",
            "ec2:DeleteNetworkInterface"
            ],
            "Resource": "*"
        }
    ]
}
```

2. Modify the Lambda function to connect to the VPC:
- Go to the "simpleLambda" function in the AWS Management Console
- Under "VPC", select the VPC you created in step 1
- Select the subnets and the security group you created for the VPC
- Click "Save" to apply the changes
- It can take a while to apply the changes

3. Test the Lambda function to ensure it's working as expected
- Use the AWS Management Console or the AWS CLI to invoke the Lambda function
- Observe the logs in CloudWatch Logs to ensure there are no errors related to VPC connectivity

4. Launch an instance in one of the private subnets with the following user data

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AVAILABILITY_ZONE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat <<EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>EC2 Metadata</title>
</head>
<body>
    <h1>EC2 Instance Metadata</h1>
    <p>Instance ID: ${INSTANCE_ID}</p>
    <p>Availability Zone: ${AVAILABILITY_ZONE}</p>
</body>
</html>
EOF

chown apache:apache /var/www/html/index.html
```

4. Update the Lambda function to interact with an EC2 instance running a web server and upload a document to Amazon S3:
- Replace the Lambda function code in the `lambda_function.py` file with the following Python code

```python
import boto3
import http.client

s3 = boto3.client('s3')

# Replace the variables below with your own values
EC2_PRIVATE_IP = '<INSTANCE_PRIVATE_IP>'
S3_BUCKET_NAME = '<BUCKET_NAME>'
S3_OBJECT_KEY = 'web-server-message.txt'

def get_web_server_message():
    conn = http.client.HTTPConnection(EC2_PRIVATE_IP)
    conn.request("GET", "/")
    response = conn.getresponse()
    data = response.read().decode("utf-8")
    conn.close()
    return data

def upload_to_s3(message):
    try:
        response = s3.put_object(Bucket=S3_BUCKET_NAME, Key=S3_OBJECT_KEY, Body=message)
        print("Document uploaded to S3:", response)
    except Exception as error:
        print("Error:", error)
        raise error

def lambda_handler(event, context):
    print("Lambda function invoked")

    try:
        message = get_web_server_message()
        print("Message from web server:", message)
        upload_to_s3(message)
        return "Document uploaded to S3 successfully!"
    except Exception as error:
        print("Error:", error)
        raise error
```

Replace `EC2_PRIVATE_IP` with the private IP address of your EC2 instance running the web server and `S3_BUCKET_NAME` with the name of the S3 bucket you created for storing the retrieved document

5. Test the updated Lambda function to ensure it retrieves the message from the EC2 instance and uploads it to the S3 bucket:
- Invoke the Lambda function using the AWS Management Console or the AWS CLI
- Verify that the document has been uploaded to the S3 bucket with the correct message


# Lab 5 - Create a simple serverless web application using AWS SAM

1. Install the SAM CLI or use AWS CloudShell
2. Initialize a new SAM application using the following command (accept defaults):

```bash
sam init --runtime python3.9 --dependency-manager pip --app-template hello-world
```

3. Change directory into the SAM app
4. Replace the template.yaml with the following code

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Simple serverless web application

Resources:
  WebFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: hello_world/
      Handler: app.lambda_handler
      Runtime: python3.9
      Events:
        Api:
          Type: Api
          Properties:
            Path: /web
            Method: get

  WebApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: Prod
      EndpointConfiguration: REGIONAL
      DefinitionBody:
        swagger: "2.0"
        info:
          title: "Serverless Web App API"
        paths:
          /web:
            get:
              x-amazon-apigateway-integration:
                httpMethod: POST
                type: aws_proxy
                uri:
                  Fn::Sub: arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${WebFunction.Arn}/invocations
              responses: {}

  WebBucket:
    Type: AWS::S3::Bucket
    Properties:
      WebsiteConfiguration:
        IndexDocument: index.html
```

5. Replace the content of the app.py file in the hello_world directory with the following Python code

```Python
import json

def lambda_handler(event, context):
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json'
        },
        'body': json.dumps({"message": "Hello from the serverless web application!"})
    }
```
6. Build the application using the following command

```bash
sam build
```

7. Package the application using the following command

```bash
sam package --output-template-file packaged.yaml --s3-bucket <your-bucket-name>
```

***you may need to edit the `samconfig.toml` file  setting "resolve_s3" to "false" in two places***

8. Run the command provided in the output to deploy the SAM application

9. Test the serverless web application
- Retrieve the API Gateway URL from the AWS Management Console or the AWS CLI
- Use a web browser or an API testing tool like Postman to send a GET request to the /web path on the API Gateway URL
- Verify that the response contains the expected message: "Hello from the serverless web application!

10. Create an S3 static website that accesses the application. Create an index.html with the following code (update the API gateway URL)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Serverless Web App</title>
</head>
<body>
    <h1>Serverless Web Application</h1>
    <p id="message">Loading message...</p>

    <script>
        async function fetchMessage() {
            const apiUrl = 'YOUR_API_GATEWAY_URL/web';
            try {
                const response = await fetch(apiUrl);
                if (response.ok) {
                    const data = await response.json();
                    document.getElementById('message').innerText = data.message;
                } else {
                    throw new Error(`HTTP error: ${response.status}`);
                }
            } catch (error) {
                console.error('Error fetching message:', error);
                document.getElementById('message').innerText = 'Error fetching message';
            }
        }

        fetchMessage();
    </script>
</body>
</html>
```

11. Open the website endpoint and check it returns the message from the serverless application

12. Delete the stack

```bash
aws cloudformation delete-stack --stack-name <your-stack-name>
```