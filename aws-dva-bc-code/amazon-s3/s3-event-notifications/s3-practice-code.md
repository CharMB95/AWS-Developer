# Lab 1 - S3 Access Control

1. Create an S3 bucket named "bootcamp-s3-exercises"

aws s3api create-bucket --bucket bootcamp-s3-exercises --region us-east-1

2. Create a new IAM user named "bootcamp-test-user" and generate access keys

aws iam create-user --user-name bootcamp-test-user
aws iam create-access-key --user-name bootcamp-test-user

3. Use 'aws configure' to create a profile for the bootcamp test user (e.g. --profile bootcamp-test-user) and use the access keys generated earlier

4. Use CloudShell and nano to create an IAM policy for restricted access named iam_policy.json

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::bootcamp-s3-exercises"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::bootcamp-s3-exercises/*"
      ]
    }
  ]
}
```

5. Attach the IAM policy to the bootcamp-test-user

aws iam put-user-policy --user-name bootcamp-test-user --policy-name BootcampTestUserPolicy --policy-document file://iam_policy.json

6. Create a bucket policy that allows the PutObject API action named bucket_policy.json

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:user/bootcamp-test-user"
      },
      "Action": [
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::bootcamp-s3-exercises",
        "arn:aws:s3:::bootcamp-s3-exercises/*"
      ]
    }
  ]
}
```

7. Try to upload, download, and delete objects using the bootcamp-test-user profile

8. Attach the bucket policy to the bootcamp-s3-exercises bucket

aws s3api put-bucket-policy --bucket bootcamp-s3-exercises --policy file://bucket_policy.json

9. Try to upload, download, and delete objects using the bootcamp-test-user profile

10. Make the object public using an object-level ACL

Which settings must be changed?
Does the bucket policy need updating?
After getting it working, what happens if you enable "block public access"?


# Lab 2 - MFA with Amazon S3

1. Enable versioning

aws s3api put-bucket-versioning --bucket bootcamp-s3-exercises --versioning-configuration Status=Enabled

2. Login as root and create access keys. Ensure you have an MFA device setup for root and make note of the ARN. Configure the access keys on the AWS CLI (not CloudShell)

3. Enable MFA delete from the CLI, not from CloudShell (update the bucket name, ARN, and replace TOKEN_CODE with a valid code)

aws s3api put-bucket-versioning --bucket <my-bucket> --versioning-configuration Status=Enabled,MFADelete=Enabled --mfa "arn:aws:iam::<account-id>:mfa/root-account-mfa-device <TOKEN_CODE>"

4. Upload and and then attempt to delete an object

echo "Test MFA Delete" > test.txt
aws s3 cp test.txt s3://bootcamp-s3-exercises/test.txt

aws s3 rm s3://bootcamp-s3-exercises/test.txt
aws s3api delete-object --bucket bootcamp-s3-exercises --key test.txt --version-id <version-id-of-deleted-object>

5. Delete the object with MFA

aws s3api delete-object --bucket bootcamp-s3-exercises --key test.txt --version-id <version-id-of-deleted-object> --mfa "arn:aws:iam::ACCOUNT_ID:mfa/root-account-mfa-device TOKEN_CODE"


# Lab 3 -  MFA-Protected API Access

1. Create a bucket

aws s3api create-bucket --bucket bootcamp-s3-mfa-test --region us-east-1

2. Update the following bucket policy with the ARN of your individual IAM account (which should have MFA enabled) and update the bucket ARNs then save as 'mfa_bucket_policy.json'

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Principal": {
                "AWS": "arn:aws:iam::676238238199:user/neal"
            },
            "Action": [
                "s3:ListBucket",
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::bootcamp-s3-mfa-test",
                "arn:aws:s3:::bootcamp-s3-mfa-test/*"
            ],
            "Condition": {
                "Null": {
                    "aws:MultiFactorAuthAge": "true"
                }
            }
        }
    ]
}
```

3. Apply the bucket policy

aws s3api put-bucket-policy --bucket bootcamp-s3-mfa-test --policy file://mfa_bucket_policy.json

4. Attempt to upload a file from the AWS CLI (not CloudShell) using the same account specified in the principal element (should fail)
5. Get temporary session credentials with MFA (update the ARN)

aws sts get-session-token --serial-number arn:aws:iam::ACCOUNT_ID:mfa/DEVICE_NAME --token-code <token>

6. Then create the environment variables (use "set" for Windows instead of "export")

export AWS_ACCESS_KEY_ID=TEMP_ACCESS_KEY_ID
export AWS_SECRET_ACCESS_KEY=TEMP_SECRET_ACCESS_KEY
export AWS_SESSION_TOKEN=TEMP_SESSION_TOKEN

7. Attempt to upload a file from the AWS CLI again, without specifying your profile (should work)

# Lab 4 - Enforce Encryption with AWS KMS

1. Update the mfa_bucket_policy.json file to enforce encryption with AWS KMS

```json
{
    "Version": "2012-10-17",
    "Id": "PutObjectPolicy",
    "Statement": [
        {
            "Sid": "DenyUnEncryptedObjectUploads",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::bootcamp-s3-mfa-test/*",
            "Condition": {
                "StringNotEquals": {
                    "s3:x-amz-server-side-encryption": "aws:kms"
                }
            }
        }
    ]
}
```

2. Update the bucket policy

aws s3api put-bucket-policy --bucket bootcamp-s3-mfa-test --policy file://mfa_bucket_policy.json

3. Test uploading a file through the console with default encryption settings (should fail)

4. Try again specifying server-side encryption with AWS KMS (should work)


# Lab 5 - S3 Event Notifications

1. Create two buckets

aws s3api create-bucket --bucket bootcamp-s3-website --region us-east-1
aws s3api create-bucket --bucket bootcamp-s3-uploads --region us-east-1 

2. Enable static website hosting on the bootcamp-s3-website bucket

aws s3 website s3://bootcamp-s3-website/ --index-document index.html

3. Create a Lambda function named `S3EventToLambda` with the following function code and the Python 3.8 environment

***python code begin***

```python
import json

def lambda_handler(event, context):
    for record in event['Records']:
        print(f"Bucket: {record['s3']['bucket']['name']}, Key: {record['s3']['object']['key']}")

    return {
        'statusCode': 200,
        'body': json.dumps('Successfully processed S3 event.')
    }
```

4. Create an event notification on the uploads bucket to trigger the Lambda function based on object creation events
5. Open the index.html file in VS code. Add the access key and secret access key of an account with permission (NOT a security best practice!! but OK for this lab)
6. Upload the index.html file to the bootcamp-s3-website bucket

aws s3 cp index.html s3://bootcamp-s3-website/index.html --acl public-read

7. Enable ACLs on the bootcamp-s3-website bucket (or add a bucket policy allowing GetObject publicly)
8. Configure CORS on the bootcamp-s3-uploads bucket to allow the static website to upload files to it (update the URL)

[
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "PUT",
            "POST",
            "DELETE"
        ],
        "AllowedOrigins": [
            "http://bootcamp-s3-website.s3-website-us-east-1.amazonaws.com"
        ],
        "ExposeHeaders": []
    }
]

9. Test uploading a file using the form on the static website

Was the object successfully saved to the bootcamp-s3-uploads bucket?
Was Lambda triggered? Is there a CloudWatch Logs event?

## HOMEWORK

Clean up all the buckets - you'll need to solve a few problems with the MFA delete protected bucket!