# Ansible and Jenkins CI Session

## Using this guide

While working with this file we will need to find and replace the values of each of the following:

- AWS Region : `REPLACE_WITH_AWS_REGION`
- Instances Stack Name: `REPLACE_WITH_INSTANCES_STACK_NAME`
- Deploy Stack Name: `REPLACE_WITH_DEPLOY_STACK_NAME`
- Key Pair Name: `REPLACE_WITH_KEY_PAIR_NAME`
- S3 Bucket Name: `REPLACE_WITH_S3_BUCKET_NAME`
- Master Instance Public IP: `REPLACE_WITH_MASTER_INSTANCE_PUBLIC_IP`
- Web Instance IP: `REPLACE_WITH_WEB_INSTANCE_IP`
- Web Instance Public IP: `REPLACE_WITH_PUBLIC_WEB_INSTANCE_IP`
- Ansible Credential ID: `REPLACE_WITH_ANSIBLE_CREDENTIALS_ID`

## Ansible

### Lab 0 : Build the environment
You can skip the key pair creation if you already have one and you have access to the pem file

#### 1. Create Key Pair
a. Login to your AWS Account

b.  Go to "EC2"

c. Under "Network and Security",  click on "Key Pairs"

d. Click "Create Key Pair" on the top right

e. Download the pem file and keep it for later use

#### 2. Deploy the Instances

a. Go to "CloudFormation"

b. Click "Create Stack" on the top right and from the drop down select "with new resources (standard)"

c. Choose "Build from Application Composer"

d. Click "Create in Application Composer Button"

e. Click on Template to switch to Template mode. Make sure YAML is selected

f. Copy the contents of this file: https://raw.githubusercontent.com/AlyIbrahim/DevOpsJourney/main/cloud-formation-templates/instances.yaml

g. Paste it in the template

h. Click "Create template"

i. Click "Confirm and continue to CloudFormation"

j. Click "Next"

k. Give a name to your stack "REPLACE_WITH_INSTANCES_STACK_NAME"

l. For "InstancesKey" Select the key pair you just created

m. Click "Next" and then "Next"

n. At the bottom, the Capabilities section Check "I acknowledge that AWS CloudFormation might create IAM resources with custom names."

o. Click "Submit"

Now Chceck when your instances have been created and connect to both of them

#### 2. Deploy the Instances (AWS CLI)

aws cloudformation create-stack --stack-name REPLACE_WITH_INSTANCES_STACK_NAME --template-body file://cloud-formation-templates/instances.yaml --parameters ParameterKey=InstancesKey,ParameterValue=REPLACE_WITH_KEY_PAIR_NAME --region REPLACE_WITH_AWS_REGION --capabilities CAPABILITY_NAMED_IAM

aws cloudformation create-stack --stack-name REPLACE_WITH_DEPLOY_STACK_NAME --template-body file://cloud-formation-templates/devops.yaml --parameters ParameterKey=S3BucketName,ParameterValue=REPLACE_WITH_S3_BUCKET_NAME --region REPLACE_WITH_AWS_REGION --capabilities CAPABILITY_IAM


#### 3. Deploy The Cloud Deploy Application

Use the same steps for deploying the instances Section 2, but use the stack from this url: https://raw.githubusercontent.com/AlyIbrahim/DevOpsJourney/main/cloud-formation-templates/devops.yaml

### Lab 1.1 : Ansible Configuration

**On the Master Instance**

a. Ssh to the web machine using ansible user
`ssh ansible@REPLACE_WITH_WEB_INSTANCE_IP`

b. Type yes to store the finger print of the web instance to the master known hosts

c. Clone the git repo
`git clone https://github.com/AlyIbrahim/DevOpsJourney`

d. Update the inventory file
`sed -i 's/WEB_INSTANCE_IP/REPLACE_WITH_WEB_INSTANCE_IP/g' DevOpsJourney/ansible-playbooks/inventory.cfg `

e. Copy the file to be the default ansible inventory
`sudo cp DevOpsJourney/ansible-playbooks/inventory.cfg /etc/ansible/hosts`

f. Update ansible configuration file to disable host checking
`echo -e "[defaults]\nhost_key_checking = False" > /etc/ansible/ansible.cfg`

### Lab 1.2 : Using Ansible Ad-Hoc

a. Let's ping our servers to check if they are alive

`ansible all -m ping`

b. Let's check the web instance facts
`ansible web -m ansible.builtin.setup`

c. Let's install Nginx on the Web instance

`ansible web -i DevOpsJourney/ansible-playbooks/inventory.cfg -m yum -a 'name=nginx state=latest' -b`

d. Start the Nginx server
`ansible web -i DevOpsJourney/ansible-playbooks/inventory.cfg -m service -a 'name=nginx state=started' -b`

e. Use your browser to check if nginx is up and running
In your browser go to: `http://REPLACE_WITH_PUBLIC_WEB_INSTANCE_IP`

### Lab 2: Using Ansible Playbooks

a. Install Nginx using the playbook

`ansible-playbook DevOpsJourney/ansible-playbooks/nginx-playbook.yaml -b`

Note: Nothing has changed ..  It is already installed

b. Install NodeJS on both instances

`ansible-playbook DevOpsJourney/ansible-playbooks/node-playbook.yaml -b`

c. Let's write a playbook to create few files
`ansible-playbook DevOpsJourney/ansible-playbooks/files-playbook.yaml`

d. Now, let's update our website
`ansible-playbook DevOpsJourney/ansible-playbooks/site-playbook.yaml`

e. Finally, let's Install Jenkins
`ansible-playbook DevOpsJourney/ansible-playbooks/jenkins-playbook.yaml -b`

Open Jenkins from your browser with address: `http://REPLACE_WITH_MASTER_INSTANCE_PUBLIC_IP:8080`

Copy the printed out default admin password from the ansible playbook and paste it to login ..

## Jenkins

### Lab 3: Jenkins Configurations

a. Paste the initial admin password printed out in the previous step.

#### 1. Initial Configuration
 
Select "Install default plugins"

Create an admin user

Click "Next" to set Jenkins URL

Click "Start using Jenkins"

#### 2.Install Additions Plugins

Click on "Manage Jenkins" on the left menu

Click on Plugins

Available Plugins from the left menu

Search and select:
- Ansible
- AWS Credentials
- S3 publisher
- AWS CodeDeploy


- Amazon EC2
- AWS Codebuild
- AWS Codebuild Cloud

Click "Install"

At the bottom of the page check "Restart Jenkins when installation is complete and no jobs are running"

Click on "Dashboard" and wait for Jenkins to restart

#### 3.System Configuration

Sign in with the admin user you created earlier

Click on "Manage Jenkins" on the left menu

Click on "System"

Scroll to the bottom, and find "S3 Profiles", Click "Add"

Enter Profile name:  `S3DevOps`

Check "Use IAM Role"

Click "Save"

#### 4.Credentials Configuration

Click on "Manage Jenkins" on the left menu

Click on "Creadentials"

Under The section "Stores scoped to Jenkins", Click on the (global) domain

On the top right click "Add Credentials"

1- Ansible

Choose Kind "Username and Password"
username: `ansible`
password: `ansible`

Click Save
Copy the Credentials ID

Find and Replace REPLACE_WITH_ANSIBLE_CREDENTIALS_ID with the created Credentials ID



#### 5.Node Configuration

Click on "Manage Jenkins" on the left menu

Click on "Nodes"

Click on "Built-In Node"

Click on "Configure" from the left Menu

in the Labels field enter: `nodejs`

Select "Environment variables", then Click "Add"
Enter the following:
Name: `WEB_INSTANCE_IP`
Value: `REPLACE_WITH_WEB_INSTANCE_IP`

Click "Save"

### Lab 4: Create a Freestyle project

Click on the "Dashboard" on the top left

Click on "+ New Item"

Enter the item name: `SimpleNodeFreestyle`

Select "Freestyle project" and click "OK"

Scroll to "Source Code Management"

Enter Repository URL: `https://github.com/AlyIbrahim/DevOpsJourney`
Branch Specifier: `*/main`

In "Build Triggers"

Check "Poll SCM"

Enter Schedule: `H/2 * * * *`

This will check the SCM every 2 mins

In section "Build Environment":

Check "Delete workspace before build starts"

In section Build Steps:

Click "Add Build Step" and select "Execute shell"

Enter the following in the "Command" box:

```
echo 'Building..'
cd simple-node
pwd
npm install
```

Click "Add Build Step" and select "Execute shell"

Enter the following in the "Command" box:
```
rm -rf app.zip
cd simple-node
zip -r ../app.zip .
```
**Optional Step **
In section Post-Build Actions:

Click "Add post-build Action " and select "Publish artifacts to S3 Bucket"

Select S3 Profile `S3DevOps`
Files to Upload:
Source: `app.zip`
Destination Bucket: `REPLACE_WITH_S3_BUCKET_NAME/build`
Bucket Region: `REPLACE_WITH_AWS_REGION`


Click "Add post-build Action " and select "Deploy an application to AWS CodeDeploy"

AWS CodeDeploy Application Name: `SimpleNode`
AWS CodeDeploy Deployment Group: `GroupOne`
AWS Region: `US_WEST_2`
S3 Bucket: `REPLACE_WITH_S3_BUCKET_NAME`
S3 Prefix: `build`
Subdirectory: `simple-node`

### Lab 5: Create a Pipeline project

`sed -i 's/MY_WEB_INSTANCE_IP/REPLACE_WITH_WEB_INSTANCE_IP/g' DevOpsJourney/simple-node/Jenkinsfile`

`sed -i 's/MY_ANSIBLE_CREDENTIALS_ID/REPLACE_WITH_ANSIBLE_CREDENTIALS_ID/g' DevOpsJourney/simple-node/Jenkinsfile`

`sed -i 's/MY_S3_BUCKET_NAME/REPLACE_WITH_S3_BUCKET_NAME/g' DevOpsJourney/simple-node/Jenkinsfile`


Click on "+ New Item"

Enter item name: SimpleNode1
Select Pipeline
Click OK

Scroll down to Script
Add the Jenkins file contents


On the lef Menu Click "Build Now"

On the bottom left, Click on the newly create number in Build History Widget
Click on "Console Output" on the left menu