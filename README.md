# Bastion_Vpc_achitecture
![image](https://github.com/user-attachments/assets/754aa0c3-78d9-4c6e-8b8c-48c88e89c31a)


Step 1: Launch Base EC2 Instance

Navigate to EC2 Dashboard

Go to AWS Console → Services → EC2
Click "Launch Instance"


Configure Instance

Name: Golden-AMI-Base
AMI: Select Amazon Linux 2 AMI (HVM)
Instance Type: t2.micro
Key Pair: Select your existing key pair or create new one
Network: Use default VPC for now
Security Group: Create new or select existing with SSH access
Storage: 8 GB gp2 (default)
Click "Launch Instance"



Step 2: Configure the Golden AMI Instance

Connect to Instance

Select your instance → Connect → SSH client
Use the provided SSH command to connect


Install Required Software

Step 3: Create Golden AMI

Stop the Instance

EC2 Dashboard → Instances → Select your instance
Actions → Instance State → Stop Instance


Create AMI

With instance selected → Actions → Image and templates → Create image
Image name: Golden-AMI-WebServer
Image description: Golden AMI with Apache, Git, CloudWatch Agent, SSM Agent
Click "Create image"
Note the AMI ID - you'll need this later!


Step 1: Create Bastion VPC (192.168.0.0/16)

Create VPC

VPC Dashboard → Your VPCs → Create VPC
Resources to create: VPC only
Name: Bastion-VPC
IPv4 CIDR: 192.168.0.0/16
IPv6 CIDR block: No IPv6 CIDR block
Tenancy: Default
Click "Create VPC"


Enable DNS Hostnames

Select your Bastion-VPC → Actions → Edit VPC settings
Check "Enable DNS hostnames"
Save changes


Create Public Subnet for Bastion

Subnets → Create subnet
VPC ID: Select Bastion-VPC
Subnet name: Bastion-Public-Subnet
Availability Zone: us-east-1a
IPv4 CIDR block: 192.168.1.0/24
Click "Create subnet"



Step 2: Create Application VPC (172.32.0.0/16)

Create Application VPC

VPC Dashboard → Your VPCs → Create VPC
Name: Application-VPC
IPv4 CIDR: 172.32.0.0/16
Click "Create VPC"


Enable DNS Hostnames

Select Application-VPC → Actions → Edit VPC settings
Check "Enable DNS hostnames"
Save changes


Create Subnets for Application VPC
Public Subnet 1a:

Subnets → Create subnet
VPC ID: Application-VPC
Subnet name: App-Public-Subnet-1a
Availability Zone: us-east-1a
IPv4 CIDR block: 172.32.1.0/24

Public Subnet 1b:

Subnet name: App-Public-Subnet-1b
Availability Zone: us-east-1b
IPv4 CIDR block: 172.32.2.0/24

Private Subnet 1a:

Subnet name: App-Private-Subnet-1a
Availability Zone: us-east-1a
IPv4 CIDR block: 172.32.10.0/24

Private Subnet 1b:

Subnet name: App-Private-Subnet-1b
Availability Zone: us-east-1b
IPv4 CIDR block: 172.32.20.0/24

Click "Create subnet" for each

Step 3: Create Internet Gateways

Create IGW for Bastion VPC

Internet Gateways → Create internet gateway
Name: Bastion-IGW
Click "Create internet gateway"
Select the IGW → Actions → Attach to VPC
Select Bastion-VPC → Attach internet gateway


Create IGW for Application VPC

Internet Gateways → Create internet gateway
Name: App-IGW
Click "Create internet gateway"
Attach to Application-VPC



Step 4: Create NAT Gateway

Allocate Elastic IP

EC2 Dashboard → Network & Security → Elastic IPs
Allocate Elastic IP address → Allocate


Create NAT Gateway

VPC Dashboard → NAT Gateways → Create NAT Gateway
Name: App-NAT-Gateway
Subnet: App-Public-Subnet-1a
Connectivity type: Public
Elastic IP allocation ID: Select the EIP you just created
Click "Create NAT Gateway"



Step 5: Create Route Tables

Create Route Table for Bastion Public Subnet

Route Tables → Create route table
Name: Bastion-Public-RT
VPC: Bastion-VPC
Click "Create route table"

Add Route:

Select the route table → Routes tab → Edit routes
Add route: Destination 0.0.0.0/0, Target: Bastion-IGW
Save changes

Associate Subnet:

Subnet Associations tab → Edit subnet associations
Select Bastion-Public-Subnet → Save associations


Create Route Tables for Application VPC
Public Route Table:

Route Tables → Create route table
Name: App-Public-RT
VPC: Application-VPC
Add route: 0.0.0.0/0 → App-IGW
Associate with App-Public-Subnet-1a and App-Public-Subnet-1b

Private Route Table:

Route Tables → Create route table
Name: App-Private-RT
VPC: Application-VPC
Add route: 0.0.0.0/0 → App-NAT-Gateway (select from NAT Gateway list)
Associate with App-Private-Subnet-1a and App-Private-Subnet-1b



Step 6: Create Transit Gateway

Create Transit Gateway

VPC Dashboard → Transit Gateways → Create Transit Gateway
Name: Multi-VPC-TGW
Description: Multi-VPC Transit Gateway
Amazon side ASN: 64512 (default)
Auto accept shared attachments: Enable
Default route table association: Enable
Default route table propagation: Enable
Click "Create Transit Gateway"


Attach VPCs to Transit Gateway
Attach Bastion VPC:

Transit Gateway Attachments → Create Transit Gateway Attachment
Transit Gateway ID: Select Multi-VPC-TGW
Attachment type: VPC
VPC ID: Bastion-VPC
Subnet IDs: Bastion-Public-Subnet
Name: Bastion-TGW-Attachment
Click "Create attachment"

Attach Application VPC:

Create another attachment
VPC ID: Application-VPC
Subnet IDs: App-Private-Subnet-1a
Name: App-TGW-Attachment


Update Route Tables for Inter-VPC Communication

Go to Route Tables
Edit Bastion-Public-RT: Add route 172.32.0.0/16 → Multi-VPC-TGW
Edit App-Private-RT: Add route 192.168.0.0/16 → Multi-VPC-TGW



Step 7: Create CloudWatch Log Groups

Create Log Group

CloudWatch Dashboard → Logs → Log groups
Create log group
Log group name: /aws/vpc/flowlogs
Click "Create"


Create Log Streams

Select the log group → Actions → Create log stream
Log stream name: bastion-vpc-flowlogs
Create another: application-vpc-flowlogs



Step 8: Create IAM Role for VPC Flow Logs

Create Role

IAM Dashboard → Roles → Create role
Trusted entity type: AWS service
Use case: VPC - Flow Logs
Click "Next"


Attach Policy

Search and select: flowlogsDeliveryRolePolicy
Click "Next"


Role Details

Role name: flowlogsRole
Click "Create role"



Step 9: Enable VPC Flow Logs

Enable for Bastion VPC

VPC Dashboard → Your VPCs → Select Bastion-VPC
Flow logs tab → Create flow log
Filter: All
Destination: Send to CloudWatch Logs
Destination log group: /aws/vpc/flowlogs
IAM role: flowlogsRole
Log record format: AWS default format
Click "Create flow log"


Enable for Application VPC

Repeat the same process for Application-VPC



Step 10: Create Security Groups

Security Group for Bastion Host

EC2 Dashboard → Security Groups → Create security group
Security group name: bastion-sg
Description: Security group for bastion host
VPC: Bastion-VPC

Inbound Rules:

Type: SSH, Protocol: TCP, Port: 22, Source: 0.0.0.0/0
Click "Create security group"


Security Group for Application Servers

Create security group
Security group name: app-sg
Description: Security group for application servers
VPC: Application-VPC

Inbound Rules:

Type: SSH, Protocol: TCP, Port: 22, Source: bastion-sg (search and select)
Type: HTTP, Protocol: TCP, Port: 80, Source: 0.0.0.0/0
Click "Create security group"



Step 11: Deploy Bastion Host

Allocate Elastic IP for Bastion

EC2 Dashboard → Elastic IPs → Allocate Elastic IP address


Launch Bastion Instance

EC2 Dashboard → Launch Instance
Name: Bastion-Host
AMI: Amazon Linux 2
Instance type: t2.micro
Key pair: Select your key pair
Network settings: Edit

VPC: Bastion-VPC
Subnet: Bastion-Public-Subnet
Auto-assign public IP: Enable
Security group: bastion-sg


Click "Launch instance"


Associate Elastic IP

Select Bastion instance → Actions → Networking → Associate Elastic IP address
Select the allocated EIP → Associate



Step 12: Create S3 Bucket

Create Bucket

S3 Dashboard → Create bucket
Bucket name: my-app-config-[random-number] (must be globally unique)
Region: Same as your VPC region
Keep other settings as defaults
Click "Create bucket"


Upload Configuration File

Select your bucket → Upload
Create a simple text file named app-config.conf with sample content
Upload the file



Step 13: Create IAM Role for EC2 Instances

Create Role

IAM Dashboard → Roles → Create role
Trusted entity: AWS service
Use case: EC2
Click "Next"


Attach Policies

Search and attach: AmazonSSMManagedInstanceCore
Click "Next"


Create Custom S3 Policy

Click "Create policy" (opens new tab)
Service: S3
Actions: GetObject, ListBucket
Resources:

Bucket: arn:aws:s3:::your-bucket-name
Object: arn:aws:s3:::your-bucket-name/*


Click "Next" → Name: S3ConfigAccess → Create policy


Complete Role Creation

Back to role creation, refresh and attach S3ConfigAccess
Role name: EC2-SSM-S3-Role
Click "Create role"



Step 14: Create Launch Template

Create Launch Template

EC2 Dashboard → Launch Templates → Create launch template
Launch template name: app-launch-template
Template version description: v1


Configure Template

AMI: Select your Golden-AMI-WebServer (from Step 3)
Instance type: t2.micro
Key pair: Select your key pair
Security groups: app-sg
Advanced details:

IAM instance profile: EC2-SSM-S3-Role
User data:

bash#!/bin/bash
yum update -y
systemctl start httpd
systemctl enable httpd

# Create sample web page
echo "<h1>Hello from Auto Scaling Group!</h1>" > /var/www/html/index.html
echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html

# Download config from S3 (replace with your bucket name)
aws s3 cp s3://your-bucket-name/app-config.conf /etc/app-config.conf

# Start CloudWatch Agent
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
    -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s

Click "Create launch template"



Step 15: Create Auto Scaling Group

Create Auto Scaling Group

EC2 Dashboard → Auto Scaling Groups → Create Auto Scaling group
Auto Scaling group name: app-asg
Launch template: app-launch-template
Click "Next"


Configure Settings

VPC: Application-VPC
Subnets: Select App-Private-Subnet-1a and App-Private-Subnet-1b
Click "Next"


Configure Advanced Options

Load balancing: Attach to a new load balancer
Load balancer type: Network Load Balancer
Load balancer name: app-nlb
Scheme: Internet-facing
Network mapping: Select App-Public-Subnet-1a and App-Public-Subnet-1b
Listeners and routing: Create a target group

Target group name: app-target-group
Protocol: TCP
Port: 80


Health checks: ELB
Click "Next"


Configure Group Size

Desired capacity: 2
Minimum capacity: 2
Maximum capacity: 4
Scaling policies: None (for now)
Click "Next"


Add Tags

Key: Name, Value: app-asg
Click "Next" → "Create Auto Scaling group"



Step 16: Verify Network Load Balancer

Check Load Balancer

EC2 Dashboard → Load Balancers
Find your app-nlb
Note the DNS name - you'll use this to access your application


Check Target Group Health

Target Groups → app-target-group
Registered targets tab → Wait for instances to show "healthy"
