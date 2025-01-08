## Lab 1: Resources and Methods

Create a basic API with a `/hello` resource and a GET method that invokes an AWS Lambda function, returning a static message. Python will be the language of choice for the AWS Lambda function. We will use `curl` for testing our API.

### Prerequisites
- AWS CLI installed and configured with necessary permissions.
- Basic knowledge of Python and terminal/command line usage.
- `curl` installed on your local machine for testing the API.

### Step 1: Create the AWS Lambda Function

1. Create an AWS Lambda function using the console.
2. For the name call it `HelloWorldFunction`.
3. Use the `Python 3.12` runtime.
4. The default "Hello from Lambda!" code will be used for this lab.

### Step 2: Create the API in Amazon API Gateway

#### 2.1 Create a New API

Use the AWS Management Console or AWS CLI to create a new REST API in Amazon API Gateway.

1. Navigate to Amazon API Gateway.
2. Choose **Create API**.
3. Select **REST API** and click on **Build**.
4. Enter an API name (e.g., `HelloWorldAPI`) and endpoint type (e.g., `Regional`), then click on **Create API**.

#### 2.2 Create a Resource

Create a new resource under your API named `/hello`.

1. Select your API.
2. In the Actions dropdown, select **Create Resource**.
3. Enter `hello` as the resource name and click on **Create Resource**.

#### 2.3 Create a GET Method

Create a GET method for the `/hello` resource that invokes the Lambda function.

1. Select the `/hello` resource.
2. In the Actions dropdown, select **Create Method** and choose GET.
3. Choose Lambda Function for the integration type.
4. Select the region and enter the name of the Lambda function created earlier (`HelloLambdaFunction`).
5. Click on **Save** and grant permissions when prompted.

#### 2.4 Deploy the API

Deploy your API to a new stage named `dev`.

1. In the Actions dropdown, select **Deploy API**.
2. Select **[New Stage]** in the Deployment stage.
3. Enter `dev` as the Stage name and click on **Deploy**.

### Step 3: Test the API

Use `curl` to send a GET request to your new API endpoint.

```sh
curl https://API_ID.execute-api.YOUR_REGION.amazonaws.com/dev/hello
```

- Replace `API_ID` and `YOUR_REGION` with your actual API ID and region.

You should receive a response similar to:

```json
"Hello from Lambda!!"
```

## Lab 2: Integration Types

We will extend our work from lab 1 by demonstrating two different integration types with Amazon API Gateway: Lambda Function and HTTP Endpoint. This lab helps to understand how API Gateway can serve as a versatile front door to various backend services.

### Prerequisites
- Completion of lab 1 or an existing API Gateway setup.
- Basic knowledge of AWS Lambda and HTTP protocols.

### Step 1: Lambda Integration

We'll modify the existing `/hello` resource to integrate with a Lambda function that returns dynamic content. This step assumes you have completed lab 1.

Update our Lambda function to return dynamic content. Modify `lambda_function.py` to include the current time in the response:

```python
import json
from datetime import datetime

def lambda_handler(event, context):
    current_time = datetime.now().strftime("%H:%M:%S")
    message = f"Hello, World! The current time is {current_time}."
    return {
        'statusCode': 200,
        'body': json.dumps(message)
    }
```

### Step 2: HTTP Integration

In this part, we will add a new resource to our API that integrates with an external HTTP endpoint. This demonstrates how API Gateway can route requests to HTTP-based services.

#### 2.1 Create a New Resource

Create a new resource under your API, named `/external-api`.

1. Select your API.
2. In the Actions dropdown, select **Create Resource**.
3. Enter `external-api` as the resource name and click on **Create Resource**.


#### 2.2 Create a GET Method with HTTP Integration

Set up the `/external-api` resource to perform an HTTP GET request to a public API. We'll use the JSONPlaceholder Typicode as an example (https://jsonplaceholder.typicode.com/posts/1).

1. Select the `/external-api` resource.
2. In the Actions dropdown, select **Create Method** and choose GET.
3. For the integration type, select HTTP.
4. In the Endpoint URL, enter `https://jsonplaceholder.typicode.com/posts/1`.
5. Click on **Create method**.

#### 2.3 Deploy the API (Update)

Deploy your changes to the `dev` stage to make the new resource accessible.

1. In the Actions dropdown, select **Deploy API**.
2. Choose the `dev` stage and click on **Deploy**.

### Step 3: Test the API

#### 3.1 Test Lambda Integration

Use `curl` to test the `/hello` endpoint:

```sh
curl https://API_ID.execute-api.YOUR_REGION.amazonaws.com/dev/hello
```

You should receive a dynamic response including the current time.

#### 3.2 Test HTTP Integration

Use `curl` to test the `/external-api` endpoint:

```sh
curl https://API_ID.execute-api.YOUR_REGION.amazonaws.com/dev/external-api
```

You should receive a response forwarded from the JSONPlaceholder Typicode API.


## Lab 3: Access Control with a Lambda Authorizer

In this lab we will implement a custom authorization mechanism using a Lambda authorizer (formerly known as a custom authorizer). This Lambda function will check for a valid API key in the request header and allow or deny access based on its presence and validity. This lab will teach you how to secure your APIs using token-based authorization patterns.

### Prerequisites
- Completion of lab 2 or an existing API with a Lambda function and an HTTP endpoint.
- Basic knowledge of Python and AWS Lambda.

### Step 1: Create the Lambda Authorizer Function

First, we need to create a new Lambda function that will act as our authorizer.

1. Create an AWS Lambda function using the console.
2. For the name call it `APIGatewayAuthorizer`.
3. Use the `Python 3.12` runtime.

Add the following code:

```python
import json

def lambda_handler(event, context):
    token = event['authorizationToken']
    method_arn = event['methodArn']
    
    # Simulate token validation
    if token == "allow":
        policy = generate_policy('user', 'Allow', method_arn)
    else:
        policy = generate_policy('user', 'Deny', method_arn)
    
    return policy

def generate_policy(principal_id, effect, method_arn):
    auth_response = {}
    auth_response['principalId'] = principal_id
    if effect and method_arn:
        policy_document = {
            'Version': '2012-10-17',
            'Statement': [{
                'Action': 'execute-api:Invoke',
                'Effect': effect,
                'Resource': method_arn
            }]
        }
        auth_response['policyDocument'] = policy_document
    return auth_response
```

This function checks if the provided `authorizationToken` is exactly "allow". In a real-world scenario, you would replace this logic with actual token validation (e.g., against a database of valid tokens, OAuth, JWT validation, etc.).

### Step 2: Attach the Lambda Authorizer to Your API

#### 2.1 Create the Authorizer in API Gateway

- **AWS Management Console:**
  1. Navigate to your API in the Amazon API Gateway console.
  2. In the left navigation pane, select **Authorizers**.
  3. Click **Create New Authorizer**.
  4. Enter an authorizer name (e.g., `MyLambdaAuthorizer`).
  5. Select **Lambda** for the type.
  6. Choose the Lambda function you created (`APIGatewayAuthorizer`).
  7. Enter `authorizationToken` for the "Token source" under "Lambda Event Payload".
  8. Click **Create authorizer**.

#### 2.2 Associate the Authorizer with an API Method

Choose an API method (e.g., the `/hello` GET method) to protect with the authorizer:

1. Select the method in the API Gateway console.
2. In the Method Request pane, set the Authorization dropdown to the name of your Lambda authorizer (`MyLambdaAuthorizer`).
3. Click ***Save***.

### Step 3: Deploy the API

Deploy your changes to the `dev` stage to apply the authorizer settings:

1. In the Actions dropdown, select **Deploy API**.
2. Choose the `dev` stage and click on **Deploy**.

### Step 4: Test the API with Authorization

#### 4.1 Test with a Valid Token

Use `curl` to send a request with a valid authorization token:

```sh
curl -H "authorizationToken: allow" https://API_ID.execute-api.YOUR_REGION.amazonaws.com/dev/hello
```

You should receive the expected response from the `/hello` resource.

#### 4.2 Test with an Invalid Token

Use `curl` to send a request with an invalid authorization token:

```sh
curl -H "authorizationToken: deny" https://API_ID.execute-api.YOUR_REGION.amazonaws.com/dev/hello
```

You should receive a `403 Forbidden` response, indicating that the Lambda authorizer successfully denied access.

This concludes lab 3, demonstrating how to secure your APIs with Lambda authorizers for token-based access control. This method allows you to implement custom authentication logic that can validate tokens against virtually any securityrequirement your application has.


## Lab 4: Using Request Validators

In this lab we will demonstrate how to enforce request validation in Amazon API Gateway to ensure that incoming requests to your API meet specific criteria before they are processed by your backend services. This feature helps in filtering out invalid requests, improving the robustness and security of your APIs.

### Prerequisites
- Completion of lab 3 or an existing API with a Lambda function, an HTTP endpoint, and a configured Lambda authorizer.
- Basic understanding of JSON schema.

### Step 1: Update the Lambda function code

Updating the `HelloWorldFunction` function with the following code:

```python
import json

def lambda_handler(event, context):
    # Ensure the body is parsed correctly as JSON
    try:
        body = json.loads(event.get("body", "{}"))
    except json.JSONDecodeError:
        body = {}  # Fallback to an empty dict if parsing fails
    
    # Extract the name and age with proper fallbacks
    name = body.get("name", "Unknown")
    age = body.get("age", "Unknown")
    
    # Create a response message
    message = f"Received data for {name} with age {age}."
    
    # Return a successful response with the message
    return {
        'statusCode': 200,
        'body': json.dumps({'message': message})
    }
```

This function simply reads the incoming request body, extracts the name and age, and returns a message confirming the received data.

### Step 2: Define a Model for Request Validation

First, we'll define a model for validating incoming requests. This model will specify the structure and data type requirements for request payloads.

#### 2.1 Create a Model in API Gateway

For this example, let's assume we want to validate requests to a new resource, `/data`, which expects a POST request with a JSON payload containing a `name` (string) and `age` (integer).

1. Navigate to your API in the Amazon API Gateway console.
2. In the left navigation pane, select **Models**.
3. Click **Create** and enter the following details:
    - Model Name: `DataModel`
    - Content Type: `application/json`
    - Model Schema:

```json
{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "title": "DataModel",
    "type": "object",
    "properties": {
    "name": { "type": "string" },
    "age": { "type": "integer" }
    },
    "required": ["name", "age"]
}
```
  4. Click **Create**.

### Step 3: Create a New Resource and Method

#### 3.1 Create the `/data` Resource

1. Select your API.
2. In the Actions dropdown, select **Create Resource**.
3. Enter `data` as the resource name and click on **Create Resource**.

#### 3.2 Create a POST Method for the `/data` Resource

1. Select the `/data` resource.
2. In the Actions dropdown, select **Create Method** and choose POST.
3. Set up the POST method to integrate with your backend Lambda function - use a proxy integration.
4. In the Integration Request section, select the Lambda Function you want to use.

### Step 4: Enable Request Validation

#### 4.1 Attach the Request Validator to the POST Method

1. Navigate to the `/data` POST method.
2. In the Method Request pane, find the **Request Validator** dropdown and select **Validate body, query string parameters, and headers**.
3. In the Request Body section, select the `DataModel` model you created for content type `application/json`.
4. Click the checkmark or **Save** to apply these settings.

### Step 5: Deploy the API

Deploy your changes to make the new resource and request validation active.

1. In the Actions dropdown, select **Deploy API**.
2. Choose your deployment stage (e.g., `dev`) and click on **Deploy**.

### Step 6: Test the Request Validation

#### 6.1 Test with a Valid Payload

Use `curl` to send a valid request:

```sh
curl -X POST https://API_ID.execute-api.YOUR_REGION.amazonaws.com/dev/data \
-H "Content-Type: application/json" \
-H "authorizationToken: allow" \
-d '{"name": "John Doe", "age": 30}'
```

This request should succeed since it meets the validation criteria defined in `DataModel`.

#### 6.2 Test with an Invalid Payload

Now, send an invalid request (e.g., missing the `age` field):

```sh
curl -X POST https://API_ID.execute-api.YOUR_REGION.amazonaws.com/dev/data \
-H "Content-Type: application/json" \
-H "authorizationToken: allow" \
-d '{"name": "Jane Doe"}'
```

This request should be blocked by API Gateway with a message indicating that the request validation failed because the `age` field is missing.

This concludes lab 4, demonstrating how to use request validators in Amazon API Gateway to ensure that incoming requests conform to predefined schemas. Request validation is a powerful feature for API robustness, allowing you to catch and reject incorrect or incomplete requests before they reach your backend services.


## Lab 5: Using Mapping Templates with Non-Proxy Integration

In this lab we'll create a scenario where we use API Gateway to transform an incoming request before it's sent to a Lambda function and to transform the Lambda's response before it's sent back to the client. We'll simulate a scenario where the API Gateway needs to adapt incoming requests to the expected format of an existing backend and then format the backend's response into a more client-friendly format.

### Step 1: Create a New Lambda Function

First, ensure you have a Lambda function that expects a simple JSON object with a `name` field and returns a greeting message. If you're adjusting from the previous proxy integration setup, you might need to slightly modify your Lambda function to directly return a string or a simple JSON object.

**Lambda Function (Non-Proxy):**
```python
import json

def lambda_handler(event, context):
    name = event.get('name', 'there')
    return {'greeting': f'Hello, {name}!'}
```

### Step 2: Setup API Gateway (Non-Proxy Integration)

#### 2.1 Create a New Resource and Method

For this lab, create a new resource, e.g., `/greet`, and a POST method within API Gateway that does not use the proxy integration.

- When setting up the integration request, choose "Lambda Function" and select the Lambda function you've prepared.

#### 2.2 Setup Request Mapping Template

- In the Integration Request section of your POST method, add a mapping template for `application/json`:

```json
{
"name": "$input.path('$.personName')"
}
```

This template transforms an incoming request with a `personName` field into the format expected by your Lambda function, which looks for a `name` field.

#### 2.3 Setup Response Mapping Template

- In the Integration Response section, add a mapping template for `application/json` that formats the Lambda function's response into a more detailed object:

```json
{
"success": true,
"message": "$input.path('$.greeting')"
}
```

This template wraps the greeting message returned by the Lambda function in a JSON object with a `success` flag and a `message` field.

### Step 3: Deploy and Test

After setting up your API Gateway method with the request and response mapping templates, deploy your API. Test the new setup with a request payload that includes the `personName` field:

```sh
curl -X POST https://API_ID.execute-api.YOUR_REGION.amazonaws.com/dev/greet \
-H "Content-Type: application/json" \
-d '{"personName": "John"}'
```

You should receive a transformed response that follows the structure defined in your response mapping template.


## Lab 6: Stages and Canary Deployments

In this lab we will explore how to manage and version your API using stages in Amazon API Gateway and how to implement canary deployments to safely introduce changes. Stages in API Gateway represent different environments (e.g., development, testing, production), allowing you to manage and test your API at various levels of development. Canary deployments enable you to roll out changes to a small percentage of your users before making them available to everyone, reducing the risk of introducing a breaking change.

### Prerequisites
- An API deployed in Amazon API Gateway from previous labs.
- Basic understanding of API versioning and environment management.

### Step 1: Create a New Stage for Your API

First, we’ll create a new stage that represents a production environment, assuming you already have a `dev` stage from previous labs.

- Go to Amazon API Gateway in the AWS Management Console.
- Select your API.
- Choose **Deploy API**.
- For Deployment stage, select **[New Stage]**.
- Enter `prod` as the stage name, which will represent your production environment.
- Click **Deploy**.

### Step 2: Set Up a Canary Deployment

Next, we’ll configure a canary release for the `prod` stage to safely introduce changes.

#### 2.1 Enable Canary Settings

- In the Amazon API Gateway console, navigate to your API and then to the `prod` stage.
- Select the **Canary** tab.
- Click **Create canary** to enable canary settings for this stage.
- Configure the canary settings:
  - **Canary Percent:** Set the value to 50% to specify the percentage of traffic that the canary version will receive.

#### 2.2 Deploy to Canary

When you make changes to your API and are ready to test them:

- Make your API changes (e.g., update an existing resource or add a new one).
- Go to the Actions dropdown menu and choose **Deploy API**.
- Select the `prod` stage.
- In the **Deployment type**, choose **Canary deployment**.
- Click **Deploy**.

This will apply your changes to the canary version of the `prod` stage, directing only the configured percentage of traffic to the new version.

### Step 3: Monitor and Promote Canary Deployment

After deploying your changes to the canary version:

- Monitor the performance and behavior of the canary release closely. Use CloudWatch Logs and custom logging in your Lambda functions to track errors, response times, and other metrics.
- If the canary deployment performs well, you can promote the canary version to serve all production traffic:
  - In the `prod` stage settings, click on the **Promote Canary** button. This will route 100% of the traffic to the canary version, making it the new production version.
- If issues arise, you can roll back by simply deploying a previous, stable version of your API to the canary.

### Step 4: Clean Up

When you no longer need the canary deployment or if you’re ready to start a new release cycle:

- Go to the `prod` stage settings in the API Gateway console.
- Select the **Canary** tab.
- Click **Delete** to remove the canary deployment settings.

