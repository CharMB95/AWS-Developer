# Lab 1 - Build a Docker image and deploy from Amazon ECR

## Admin privileges will be required on the EC2 instance, assign via an instance profile / role

1. Install Docker on EC2 and create an image

- Launch an EC2 instance (Aamzon Linux 2023 AMI)

- Install Docker
sudo yum install docker -y
sudo service docker start
sudo usermod -a -G docker ec2-user

- Create a directory and change into it:

mkdir ecs-lab
cd ecs-lab

- Create a Dockerfile and add edit:

touch Dockerfile

Add the following:

```dockerfile
FROM public.ecr.aws/docker/library/ubuntu:18.04

# Install dependencies
RUN apt-get update && \
 apt-get -y install apache2 curl jq

# Configure apache
RUN echo '. /etc/apache2/envvars' > /root/run_apache.sh && \
 echo 'mkdir -p /var/run/apache2' >> /root/run_apache.sh && \
 echo 'mkdir -p /var/lock/apache2' >> /root/run_apache.sh && \ 
 echo '/usr/sbin/apache2 -D FOREGROUND' >> /root/run_apache.sh && \ 
 chmod 755 /root/run_apache.sh

# Add a new script that fetches the task ARN and writes it to the index.html
ADD fetch_and_run.sh /root/fetch_and_run.sh
RUN chmod 755 /root/fetch_and_run.sh

EXPOSE 80

CMD /root/fetch_and_run.sh
```

- Create the `fetch_and_run.sh` file in the same directory

```bash
#!/bin/bash
set -e

# Fetch task metadata
METADATA=$(curl $ECS_CONTAINER_METADATA_URI_V4/task)

# Extract task ARN
TASK_ARN=$(echo $METADATA | jq -r .TaskARN)

# Write the ARN to the index.html
echo "Hello from task: $TASK_ARN" > /var/www/html/index.html

# Run the apache server
/root/run_apache.sh
```

- Build the image and validate the image

sudo docker build -t ecs-lab .

2. Push to Amazon ECR

```bash
aws ecr create-repository --repository-name ecs-lab --image-scanning-configuration scanOnPush=true --region us-east-1
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <ACCOUNT-NUMBER>.dkr.ecr.us-east-1.amazonaws.com
docker tag ecs-lab:latest <ACCOUNT-NUMBER>.dkr.ecr.us-east-1.amazonaws.com/ecs-lab
docker push <ACCOUNT-NUMBER>.dkr.ecr.us-east-1.amazonaws.com/ecs-lab
```

3. Create a task execution role

```bash
aws iam create-role --role-name ecsExecutionRole --assume-role-policy-document file://trust-policy.json
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

- Attach the AmazonECSTaskExecutionRolePolicy managed policy

```bash
aws iam attach-role-policy --role-name ecsExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

4. Create a cluster and task

```bash
aws ecs create-cluster --cluster-name ecs-lab
aws ecs put-cluster-capacity-providers --cluster ecs-lab --capacity-providers FARGATE FARGATE_SPOT  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1,base=1
```

- Create a task definition (task-definition.json) and supply the image URI and execution role

```json
{
    "executionRoleArn": "arn:aws:iam::<ACCOUNT-NUMBER>:role/ecsExecutionRole",
    "containerDefinitions": [
        {
            "name": "ecs-lab-task",
            "image": "<ACCOUNT-NUMBER>.dkr.ecr.us-east-1.amazonaws.com/ecs-lab",
            "essential": true,
            "portMappings": [
                {
                    "containerPort": 80
                }
            ]
        }
    ],
    "requiresCompatibilities": [
        "FARGATE"
    ],
    "networkMode": "awsvpc",
    "cpu": "256",
    "memory": "512",
    "family": "ecs-lab-task"
}
```

- Register the task definition

aws ecs register-task-definition --cli-input-json file://task-definition.json

5. Launching a Task

We will now launch a task using the task definition created in the previous step.

- Run the following command to launch a task:

```bash
aws ecs run-task --cluster ecs-lab --task-definition ecs-lab-task --count 1 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[<SUBNET-ID>],securityGroups=[<SECURITY-GROUP-ID>],assignPublicIp=ENABLED}"
```

Remember to replace `ecs-lab` and `ecs-lab-task` with your actual cluster name and task definition name. Also, you need to update your subnet and security group.


# Lab 2 - Service Auto Scaling and Elastic Load Balacning

1. Create a target group

```bash
aws elbv2 create-target-group --name ecs-lab-target-group-ip --protocol HTTP --port 80 --vpc-id <VPC-ID> --target-type ip
```

2. Create a load balancer

```bash
aws elbv2 create-load-balancer --name ecs-lab-load-balancer --subnets <SUBNET-ID> <SUBNET-ID> <SUBNET-ID> <SUBNET-ID>
```

3. Create a listener

```bash
aws elbv2 create-listener --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:<ACCOUNT-ID>:loadbalancer/app/ecs-lab-load-balancer/<ELB-ID> --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:<ACCOUNT-ID>:targetgroup/ecs-lab-target-group-ip/<TARGET-GROUP-ID>
```

4. Create an ECS service

```bash
aws ecs create-service --cluster ecs-lab --service-name ecs-lab-service --task-definition ecs-lab-task:1 --desired-count 2 --network-configuration "awsvpcConfiguration={subnets=[<SUBNET-ID>,<SUBNET-ID>,<SUBNET-ID>,<SUBNET-ID>],securityGroups=[<SECURITY-GROUP-ID>],assignPublicIp=ENABLED}"
```

5. Update the ECS service

```bash
aws ecs update-service --cluster ecs-lab --service ecs-lab-service --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:<ACCOUNT-ID>:targetgroup/ecs-lab-target-group-ip/<TARGET-GROUP-ID>,containerName=ecs-lab-task,containerPort=80
```

6. Register the ECS service as a scalable target

```bash
aws application-autoscaling register-scalable-target --service-namespace ecs --resource-id service/ecs-lab/ecs-lab-service --scalable-dimension ecs:service:DesiredCount --min-capacity 1 --max-capacity 10
```

7. Put a scaling policy

```bash
aws application-autoscaling put-scaling-policy --policy-name ecs-service-scale-out-policy --service-namespace ecs --resource-id service/ecs-lab/ecs-lab-service --scalable-dimension ecs:service:DesiredCount --policy-type TargetTrackingScaling --target-tracking-scaling-policy-configuration file://target-tracking-config.json
```

```json
{
   "TargetValue": 100,
   "PredefinedMetricSpecification": {
       "PredefinedMetricType": "ALBRequestCountPerTarget",
       "ResourceLabel": "app/<load-balancer-name>/<load-balancer-id>/targetgroup/<target-group-name>/<target-group-id>"
   }
}
```

8. Generate load on the application by using the following command:

```bash
while true; do curl http://LOAD-BALANCER-DNS-NAME; done
```

# Lab 3: Deploying and Scaling Amazon EKS Clusters

In this lab, you'll learn how to deploy an Amazon EKS cluster, deploy a sample application, and scale the cluster using the Kubernetes Horizontal Pod Autoscaler (HPA).

**Step 1: Installing the required tools**

1. Install and configure the AWS CLI

2. Install [kubectl]

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

3. Install the [eksctl]) command-line tool.


```bash
# for ARM systems, set ARCH to: `arm64`, `armv6` or `armv7`
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH

curl -sLO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"

# (Optional) Verify checksum
curl -sL "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_checksums.txt" | grep $PLATFORM | sha256sum --check

tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && rm eksctl_$PLATFORM.tar.gz

sudo mv /tmp/eksctl /usr/local/bin
```

Save the above script as `eksctl.sh` then run `chmod +x eksctl.sh` and `sudo sh eksctl.sh`

**Step 2: Creating an EKS Cluster**

1. Create a new Amazon EKS cluster using the `eksctl` command:

```bash
eksctl create cluster --name my-eks-cluster --region us-east-1 --nodegroup-name my-nodegroup --node-type t2.small --nodes 3 --nodes-min 1 --nodes-max 5 --managed
```

Replace `my-eks-cluster` and `us-east-1` with your preferred cluster name and AWS region.

**Step 3: Deploying a Sample Application**

1. Create a new file named `deployment.yaml` with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

2. Deploy the sample application to your EKS cluster:

```bash
kubectl apply -f deployment.yaml
```

**Step 4: Exposing the Application**

1. Expose the application using a Kubernetes service:

```bash
kubectl expose deployment nginx-deployment --type=LoadBalancer --name=my-service
```

2. Get the external IP address of the LoadBalancer:

```bash
kubectl get services my-service
```

You can access the sample application using the external IP address and port number.

**Step 5: Setting up Horizontal Pod Autoscaler (HPA)**

1. Create a new file named `hpa.yaml` with the following content:

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-deployment-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

2. Apply the HPA configuration:

```bash
kubectl apply -f hpa.yaml
```

Now, your EKS cluster will automatically scale the number of pods based on the CPU utilization.

3. Configure permissions for your IAM user account

```bash
kubectl get configmap aws-auth -n kube-system -o yaml > aws-auth.yaml
```
Edit the `aws-auth.yaml` file and add the `mapUsers` section (as per the below example)

```yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::<account-number>:role/eksctl-my-eks-cluster-nodegroup-m-NodeInstanceRole-ZKA8ZSWLC7Z2
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: <your-iam-user-arn>
      username: <your-iam-user-name>
      groups:
        - system:masters

kind: ConfigMap
metadata:
  creationTimestamp: "2023-05-12T10:41:25Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "1215"
  uid: 54d885a3-3484-41d4-b032-41959915d8b6
```

Then, run:

```bash
kubectl apply -f aws-auth.yaml
```

You should now have access to view all objects in the cluster

## Delete the service and cluster

kubectl delete svc my-service

eksctl delete cluster --name my-eks-cluster