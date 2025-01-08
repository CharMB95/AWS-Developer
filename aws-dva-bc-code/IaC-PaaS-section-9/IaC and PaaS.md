# Lab 1 - Create a basic CloudFormation Stack using the AWS CLI

1. Create a file named s3_bucket.yaml with the following content

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  S3Bucket:
    Type: 'AWS::S3::Bucket'
    Properties:
      BucketName: my-cloudformation-s3-bucket-3121s2
Outputs:
  BucketName:
    Description: 'The name of the created S3 bucket'
    Value: !Ref S3Bucket
```

2. Run the following command in your terminal to create the stack

aws cloudformation create-stack --stack-name MyFirstStack --template-body file://s3_bucket.yaml

3. Check the status of your stack creation by running

aws cloudformation describe-stacks --stack-name MyFirstStack

4. The stack can be deleted with the following command (don't do it yet as we'll be using this bucket)

aws cloudformation delete-stack --stack-name MyFirstStack

# Lab 2 - Working with Nested Stacks

1. Create a file named vpc.yaml with the following content

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: NestedStackVPC
Outputs:
  VpcId:
    Description: VPC ID
    Value: !Ref VPC
```

2. Create a file named subnet1.yaml with the following content

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  VpcId:
    Type: String
Resources:
  Subnet1:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      CidrBlock: 10.0.1.0/24
      VpcId: !Ref VpcId
      Tags:
        - Key: Name
          Value: NestedStackSubnet1
Outputs:
  Subnet1Id:
    Description: Subnet 1 ID
    Value: !Ref Subnet1
```

3. Create a file named subnet2.yaml with the following content

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Parameters:
  VpcId:
    Type: String
Resources:
  Subnet2:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: !Select [ 1, !GetAZs '' ]
      CidrBlock: 10.0.2.0/24
      VpcId: !Ref VpcId
      Tags:
        - Key: Name
          Value: NestedStackSubnet2
Outputs:
  Subnet2Id:
    Description: Subnet 2 ID
    Value: !Ref Subnet2
```

4. Upload the vpc.yaml, subnet1.yaml, and subnet2.yaml files to the S3 bucket and retrieve the URLs

aws s3 cp <file-name> s3://<bucketname>

aws s3api list-objects --bucket my-cloudformation-s3-bucket-3121s2 --query "Contents[].{Key: Key}" --output text | awk '{ print "https://my-cloudformation-s3-bucket-3121s2.s3.amazonaws.com/" $1 }'

5. Create a file named main.yaml with the following content (replace your-bucket-name with the name of the S3 bucket where you uploaded the templates)

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  VPCStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://my-cloudformation-s3-bucket-3121s2.s3.amazonaws.com/vpc.yaml

  Subnet1Stack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://my-cloudformation-s3-bucket-3121s2.s3.amazonaws.com/subnet1.yaml
      Parameters:
        VpcId: !GetAtt VPCStack.Outputs.VpcId

  Subnet2Stack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://my-cloudformation-s3-bucket-3121s2.s3.amazonaws.com/subnet2.yaml
      Parameters:
        VpcId: !GetAtt VPCStack.Outputs.VpcId
```

7. Deploy the main stack using the AWS CloudFormation CLI

aws cloudformation create-stack --stack-name NestedStackExample --template-body file://main.yaml --capabilities CAPABILITY_NAMED_IAM

8. The stack can be deleted with the following command

aws cloudformation delete-stack --stack-name NestedStackExample

# Lab 3 (Part 1) - Using Rollback Configuration and Retain Resource

1. Create a file named rollback_example.yaml with the following content

```yaml
Resources:
  MyS3Bucket:
    DeletionPolicy: Retain
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-rollback-example-s3-bucket-22123s
```

2. Run the following command to create the stack with a rollback configuration

aws cloudformation create-stack --stack-name MyRollbackStack --template-body file://rollback_example.yaml

3. Check the status of your stack creation, and delete the stack when done

aws cloudformation describe-stacks --stack-name MyRollbackStack
aws cloudformation delete-stack --stack-name MyRollbackStack

4. Verify that the S3 bucket still exists after the stack deletion

aws s3api list-buckets --query "Buckets[?Name=='my-rollback-example-s3-bucket-22123s']"

5. Manually delete the retained S3 bucket

aws s3api delete-bucket --bucket my-rollback-example-s3-bucket-22123s

# Lab 3 (Part 2) - Use CLI to specify resources to retain

1. Deploy the https://s3.us-west-2.amazonaws.com/cloudformation-templates-us-west-2/ELBWithLockedDownAutoScaledInstances.template template and use the default VPC
2. Launch an EC2 instance into the default VPC and attach it to the same security group as the EC2 instances
3. Delete the stack through the console - it should fail due to the attachment of the EC2 instance ENI to the security group
4. Use the AWS CLI to delete the stack whilst retaining the security group

aws cloudformation delete-stack --stack-name <stack-name> --retain-resources InstanceSecurityGroup


# Lab 4 - Deploying a sample application using Elastic Beanstalk

1. Install the Elastic Beanstalk CLI and configure your AWS credentials
2. Create a directory for the sample application and navigate to it

```bash
mkdir eb-prod-app && cd eb-prod-app
```

3. Initialize the Elastic Beanstalk application

```bash
eb init -p "python-3.11" --region us-east-1
```

4. Create a simple `application.py` file with the following content

```python
from flask import Flask
application = Flask(__name__)

@application.route('/')
def hello_world():
    return 'Hello, from AWS Elastic Beanstalk!'

if __name__ == '__main__':
    application.run(debug=True)
```

5. Create a file named `requirements.txt` file to specify the dependencies of your application.

```txt
Flask==3.0.1
```

6. Deploy the application to Elastic Beanstalk

```bash
eb create eb-prod-app-env
```

7. Open the application URL in your browser


# Lab 5 - Using .ebextensions to customize the environment


1. Create a file named `.ebextensions/01_set_environment_variables.config` with the following content

```yaml
option_settings:
  aws:elasticbeanstalk:application:environment:
    MY_ENV_VAR: my-value
```

2. Modify the application.py file to display the environment variable

```Python
from flask import Flask
import os

application = Flask(__name__)

@application.route('/')
def hello_world():
    my_env_var = os.getenv('MY_ENV_VAR', 'default-value')
    return f'Hello, from AWS Elastic Beanstalk! MY_ENV_VAR={my_env_var}'

if __name__ == '__main__':
    application.run(debug=True)
```

3. Deploy the updated application

```bash
eb deploy
```

4. Check the website URL to see if the environment variable shows correctly


# Lab 6 - Enable SSL/TLS and redirection from HTTP to HTTPS

1. Use ACM to generate an SSL/TLS certificate
2. In the Elastic Beanstalk console, navigate to your environment's configuration page
3. In the "Instance traffic and scaling" section, click "Edit"
4. In the "Listeners" section, add a new secure listener by clicking "Add listener" and selecting "HTTPS" as the protocol. For the "Port", use 443
6. In the "SSL certificate ID" field, select the SSL certificate you requested or imported in step 1 from the dropdown menu
7. Click "Save" to save your changes and then "Apply". Elastic Beanstalk will update your environment and configure the load balancer to use the SSL certificate for HTTPS traffic
8. Create an Alias to the Elastic Beanstalk environment in Route 53

***test connecting to the domain name with HTTP and HTTPS (both should work)***

9. Replace the application.py with the following code

```Python
from flask import Flask, request, redirect

application = Flask(__name__)

@application.before_request
def before_request():
    # Check if the request is not secure and the app is not running locally
    if 'localhost' not in request.host and request.headers.get('X-Forwarded-Proto', 'http') != 'https':
        url = request.url.replace('http://', 'https://', 1)
        code = 301
        return redirect(url, code=code)

@application.route('/')
def hello_world():
    return "Hello, from AWS Elastic Beanstalk with SSL/TLS encryption!"

if __name__ == '__main__':
    application.run()
```

10. Run the following command to update the environment

```bash
eb deploy
```

***test connecting to the domain name with HTTP and HTTPS (HTTP should redirect)***

11. Terminate the environments once finished

eb terminate eb-prod-app-env

12. Once the environments are fully deleted, delete the application from the console