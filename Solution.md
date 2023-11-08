# Project-503 : Blog Page Application (Django) deployed on AWS Application Load Balancer with Auto Scaling, S3, Relational Database Service(RDS), VPC's Components, DynamoDB and Cloudfront with Route 53 (SOLUTION)

## Description

The Clarusway Blog Page Application aims to deploy blog application as a web application written Django Framework on AWS Cloud Infrastructure. This infrastructure has Application Load Balancer with Auto Scaling Group of Elastic Compute Cloud (EC2) Instances and Relational Database Service (RDS) on defined VPC. Also, The Cloudfront and Route 53 services are located in front of the architecture and manage the traffic in secure. User is able to upload pictures and videos on own blog page and these are kept on S3 Bucket. This architecture will be created by Firms DevOps Guy.

## Problem Solution Order

## Steps to Solution

### Step 1: Create dedicated VPC and whole components

- Create a vpc with 2 public subnets and 2 private subnets and place them in 2 AZs, each az should include 1 public and 1 private subnet.
- We'll set CIDR block as 10.90.0.0/16 for our VPC. You can choose whatever you want. This gives us more than 65000 IP address. As you remember, after we set our CIDR block of our VPC, AWS couldn't expend it, we can just add new CIDR block.

  ### VPC

  - Lets create VPC first.
    create a vpc named `aws-capstone-vpc` CIDR blok is `10.90.0.0/16`
    no ipv6 CIDR block
    tenancy: default (Aynen EC2 daki dedicate mantığına göre hazırlanan bir servis. Kendimize dedice edebiliriz bu VPC yi)
    After creating VPS, some services are created automatically along with it. For example Route table or NACL table ---> !!! SHOW MAIN ROUTE TABLE !!! This is the main route table and we are gonna use this route table as either public route table or private route table. We'll see this a couple of steps later.

  - Ok, After creating VPC, the first thing we need to do, select VPC, click `Actions` and `enable DNS hostnames` for the `aws-capstone-vpc`. If we do not enable this, a DNS host name won't be assigned to any the machines under this VPC and these machines will not be able to communicate with each other. They need a DNS for this.

  ## Subnets

  - So, now, we can create subnets. In this example, we are gonna create 2 public and 2 private subnets within 2 AZs. Each subnet's block must be covered by `10.90.0.0/16`. They are going to be /24 subnet masks. Subnet masks of Subnets can be shown on below;
    - Public Subnets
      `10.90.10.0/24` (This mask has 2 over 6 IP addresses, we will be able to use 251 of them. Why?
      1. The first address of each block is network address and the last one is broadcast address. Two addresses is reserved for this purpose
      2. Normally, I can attach totally 254 IPs for each subnet, when we create it outside. But, in AWS world, 3 more IP addresses are reserved which are
      3. 10.90.10.1 for VPC router
      4. 10.90.10.2 for DNS
      5. 10.90.10.3 for future usage.
         )
         `10.90.20.0/24`
    - Private Subnets
      `10.90.11.0/24`
      `10.90.21.0/24`
  - Go to the subnet section on left hand side and click it. Select create subnet. We are gonna create Subnets one by one to follow directions respectively on below;
    - Create a public subnet named `aws-capstone-public-subnet-1a` under the vpc aws-capstone-VPC in AZ us-east-1a with 10.90.10.0/24
    - Create a private subnet named `aws-capstone-private-subnet-1a` under the vpc aws-capstone-VPC in AZ us-east-1a with 10.90.11.0/24
    - Create a public subnet named `aws-capstone-public-subnet-1b` under the vpc aws-capstone-VPC in AZ us-east-1b with 10.90.20.0/24
    - Create a private subnet named `aws-capstone-private-subnet-1b` under the vpc aws-capstone-VPC in AZ us-east-1b with 10.90.21.0/24
    - Show the default subnet associations on the route table `aws-capstone-default-rt` (internet access even on private subnets)
  - We have said that we want the machines we put on public subnets to reach the outside. For this purpose, we need to coordinate `auto-assign IP` setting up for public subnets. After this modification, whenever we create a machine on public subnets, Public IP address will be automatically assigned on it. We don't need to worry about it anymore.
    select public subnet-1A>>>actions>>>edit subnet settings>>>>enable auto assign ip
  - I wanna ask a question? Right now, Do you think our public subnets are really public? What is the reason we call them as public subnet? The reason is connection of internet. We are gonna manage it with route tables and Internet Gateway. Lets create Internet gateway first and then create route table.

  ## Internet Gateway

  - Click Internet gateway section on left hand side. Create an internet gateway named `aws-capstone-igw`
  - This device is a virtual device that allows our public subnets to connect to the internet and at the same time to connect to our public subnets from the internet. Actually, we call it device, but this is a virtual device.
  - ATTACH the internet gateway `aws-capstone-igw` to the newly created VPC `aws-capstone-vpc`. Go to VPC and select newly created VPC and click action ---> Attach to VPC ---> Select VPC we would like to attach to

  ## Route Table

  - Go to route tables on left hand side. We have already one route table as main route table. I am gonna set it up as public route table and create a new one which is going to be private route table.
  - What is the reason to CALL some route table as public route table? If any route table has a internet connection with Internet gateway, we call it as public route table. if it hasn't any connection from outside, it calls private route table.
  - First we create a route table and give a name as `aws-capstone-private-rt` private route table.
  - Next, Lets create a public route table. We have already a route table. It comes with VPC as default. It's called as main route table. I'm gonna turn it into public route table. Lets give it a name as `aws-capstone-public-rt` and add a rule in which destination 0.0.0.0/0 (any network, any host) to target the internet gateway `aws-capstone-igw` in order to allow access to the internet.
  - Then, we need to associate our subnets with these route tables in terms of being public or private. Select the private route table, come to the subnet association subsection and add private subnets to this route table. Similarly, we will do it for public route table and public subnets.

  ## Endpoint

  - According to our project, we need to create endpoint. Because developers don't want to expose our traffic to the internet between EC2s and S3s.
  - to do this go to the endpoint section on the left hand menu
  - select endpoint
  - click create endpoint
  - Name tag : aws-capstone-s3-gw-endpoint
  - Since we will make all EC2s locate on private subnets, Select both private subnets. Select `com.amazonaws.us-east-1.s3` as service name
  - type : Gateway
  - select `aws-capstone-vpc`
  - select private route table
  - Policy `Full Access`
  - and then create
  - go route table and check the rules. There has been newly created rule seen on table.

## Break

### Step 2: Create Security Groups (ALB ---> EC2 ---> RDS)

Before we start, we need to create our security groups. Our EC2 and RDS are going to be located in Private subnet and ALB and NAT Instance are going to be located in Public subnet. In addition, RDS only accepts traffic coming from EC2 and EC2s only accepts traffic coming from ALB. NAT instance needs to open HTTP, HTTPS and SSH ports. So, Lets create security groups based on these requirements.

1. ALB Security Group
   Name : aws-capstone-alb-sg
   Description : ALB Security Group allows traffic HTTP and HTTPS ports from anywhere
   Inbound Rules
   VPC : aws-capstone-vpc
   HTTP(80) ----> anywhere
   HTTPS (443) ----> anywhere

2. EC2 Security Groups
   Name : aws-capstone-ec2-sg
   Description : EC2 Security Groups only allows traffic coming from aws-capstone-alb-sg Security Groups for HTTP and HTTPS ports. In addition, ssh port is allowed from anywhere
   VPC : aws-capstone_VPC
   Inbound Rules
   HTTP(80) ----> aws-capstone-alb-sg
   HTTPS (443) ----> aws-capstone-alb-sg
   ssh ----> anywhere

3. RDS Security Groups
   Name : aws-capstone-rds-sg
   Description : EC2 Security Groups only allows traffic coming from aws-capstone-ec2-sg Security Groups for MYSQL/Aurora port.

VPC : aws-capstone-vpc
Inbound Rules
MYSQL/Aurora(3306) ----> aws-capstone-ec2-sg

4. NAT Instance Security Group
   Name : aws-capstone-nat-sg
   Description : NAT instance Security Group allows traffic HTTP and HTTPS and SSH ports from anywhere
   Inbound Rules
   VPC : aws-capstone-vpc
   HTTP(80) ----> anywhere
   HTTPS (443) ----> anywhere
   SSH (22) ----> anywhere

## Step 3: Prepare your Github repository

- Create private project repository on your Github and clone it on your local. Copy all files and folders which are downloaded from clarusway repo under this folder. Commit and push them on your private Git hup Repo.

- Frist Create `private` github repo
- Go to `github.com`
- First create a user access token
- Select your profile picture on the to right side
- Choose `settings`
- On the left side choose `developer settings`
- Select `Personal Access Tokens` --> `Tokens (classic)`
- Click on `Generate new token`
- Click on `Generate new token (classic)`
- Enter `capstone project access` under `Note`
- Enter `7 Days` for expiration
- Select the check box `repo`
- Click `Generate token`
- Save your token somewhere safe
- Go to github.com
- Create a new repository
- Repository name: capstone
- Description: Repo for AWS Dev Ops xxxx Capstone Project
- Choose 'Private'

- In your local machine go to Desktop
- Copy the git clone URL from github.com
- At a terminal type `git clone <Clone URL>`
- If you have authentication issues, you can enter your user name and use the token created above for your password OR try git clone https://<TOKEN>@<YOUR PRIVATE REPO URL>
- Copy the files from the clarusway repo to the capstone folder on your desktop
- Go back to the terminal
- Make sure you are in the `capstone` folder
- Type the following commands
- `git add .`
- `git commit -m "initial commit"`
- `git push`
- Verify that your new repo has your new files

## Part 2 Creating parameters in SSM Parameter Store

- Go to SSM and from left-hand menu, select Parameters Store

- Click "Create Parameter"

- Create a parameter for `database master password` :

```
 `Name`         : /<youname>/capstone/password
 `Description`  : ---
 `Tier`         : Standard
 `Type`         : SecureString   (So AWS encrypts sensitive data using KMS)
 `Value`        : Clarusway1234

- Create parameter for `database username`  :

 `Name`         : /<youname>/capstone/username
 `Description`  : ---
 `Tier`         : Standard
 `Type`         : String   (No encryption)
 `Data type`    : text
 `Value`        : admin

- Create parameter for `Github TOKEN`  :

 `Name`         : /<youname>/capstone/token
 `Description`  : ---
 `Tier`         : Standard
 `Type`         : SecureString   (So AWS encrypts sensitive data using KMS)
 `Value`        : xxxxxxxxxxxxxxxxxxxx
```

### Step 3: Create RDS

Lets create our RDS instance in which users information are kept. Before we create our database, first we need to do is to create Subnet Group. We've spoon up our RDS instance in our default VPC until now. That's why we haven't come across network issue. However, we should arrange subnet group for our custom VPC. Click `subnet Groups` on the left hand menu and click `create DB Subnet Group`

```text
Name               : aws-capstone-rds-subnet-group
Description        : aws-capstone-rds-subnet-group
VPC                : aws-capstone_VPC
Add Subnets
Availability Zones : Select 2 AZ in aws-capstone-vpc
Subnets            : Select 2 Private Subnets in these subnets (x.x.11.0, x.x.21.0)
```

- Now we can launch DB instance on RDS console. Go to the RDS console and click `create database` button

```text
Choose a database creation method : Standard Create
Engine Type     : MySQL
Edition         : MySQL Community
Version         : 8.0.33 (or latest)
Templates       : Free Tier
Settings        :
    - DB instance identifier : aws-capstone-rds
    - Master username        : admin
    - Password               : Clarusway1234
DB Instance Class            : Burstable classes (includes t classes) ---> db.t2.micro
Storage                      : 20 GB and enable autoscaling(up to 40GB)
Connectivity:
    Compute                 : Don't connect to EC2
    VPC                      : aws-capstone-vpc
    Subnet Group             : aws-capstone-rds-subnet-group
    Public Access            : No (When we indicate this as No, RDS can be reached internally. This affects only requests coming from outside)
    VPC Security Groups      : Choose existing ---> aws-capstone-rds-sg
    Availability Zone        : No preference
    Additional Configuration : Database port ---> 3306
Database authentication ---> Password authentication
Additional Configuration:
    - Initial Database Name  : clarusway
    - Backup ---> Enable automatic backups
    - Backup retention period ---> 7 days
    - Select Backup Window ---> Select 03:00 (am) Duration 1 hour
    - Maintance window : Select window    ---> 04:00(am) Duration:1 hour
create instance
```

### Step 4: Create two S3 Buckets and set one of these as static website.

Go to the S3 Console and lets create two buckets.

One of them is going to be used for videos and pictures which will be uploaded on Blog page. Other S3 Bucket is going to be used for Failover scenario. This S3 Bucket is going to publish a static website.

1. Blog Website's S3 Bucket

- Click Create Bucket

```text
Bucket Name : awscapstones<name>blog
Region      : N.Virginia
Object Ownership
    - ACLs enabled
        - Bucket owner preferred
Block Public Access settings for this bucket
Block all public access : Unchecked
Other Settings are keep them as are
create bucket
```

2. S3 Bucket for failover scenario

- Click Create Bucket

```text
Bucket Name : www.clarusway.us
Region      : N.Virginia
Object Ownership
    - ACLs enabled
        - Bucket owner preferred
Block Public Access settings for this bucket
Block all public access : Unchecked
Please keep other settings as are
```

- create bucket

- Selects created `www.clarusway.us` bucket ---> Properties ---> Static website hosting

```text
Static website hosting : Enable
Hosting Type : Host a static website
Index document : index.html
save changes
```

- Select Upload and upload index.html and sorry.jpg files ---> Access control list (ACL) ---> Everyone (public Access) allow Read for object

## Break

## Step 5: Create NAT Instance in Public Subnet

Before we create our cluster. We need to launch NAT instance. Because, our EC2 created by autoscaling groups will be located in private subnet and they need to update themselves, install required files and also need to download folders and files from Github.

```text
Go to the EC2 console and click Launch instance
- Name: aws-capstone-nat-instance
- AMI: write "amzn-ami-vpc-nat-hvm" into the filter box
select NAT Instance `ami-0c65039d08fea77c7`
    - Instance Type: t2.micro
    - Key Pair - choose your key pair
    - Network : aws-capstone-vpc
    - Subnet  : aws-capstone-public-subnet-1a
    - Security Group: aws-capstone-nat-sg
    - Termination protection        : Enable
    - Other features keep them as are
```

- Click on `Launch Instance`

- select NAT instance and enable stop source/destination check

  Each EC2 instance performs source/destination checks by default. This means that the instance must be the source or destination of any traffic it sends or recieves. But NAT instance is neither the source nor the destination of the traffic. NAT instance should be able to send or receive traffic. Because it acts as a gateway.

- go to private route table and write a rule

```
Destination : 0.0.0.0/0
Target      : instance ---> Select NAT Instance
Save
```

Test your NAT instance:

- Launch a new Linux instance in the aws-capstone-private-1a subnet
- SecGrp: aws-capstone-nat-sg
- Connect to your NAT instance:
  - In a new GitBash or MacOS Terminal:
    - change to the directory where your pem key file is
    - type: `eval $(ssh-agent)`
    - type: `ssh-add <your pem file name>`
    - type: `ssh -A ec2-user@<IP of NAT Instance>`
  - When connected to the NAT instance:
    - type `ssh ec2-user@<private IP of test instance>`
  - When connected to the private instance:
    - type `curl www.google.com`
- Terminate your test instance (NOT your NAT instance!!)

## Break

- Copy the files from the clarusway repo (src folder) to the capstone folder on your desktop
- Copy the file requirements.txt to the capstone folder

- Go back to the terminal
- Make sure you are in the `capstone` folder
- Type the following commands
- `git add .`
- `git commit -m "initial commit"`
- `git push`
- Verify that your new repo has your new files

## Step 7: Write RDS database endpoint and S3 Bucket name in settings file given by Clarusway Fullstack Developer team and push your application into your own public repo on Github

```
- Examine the existing settings.py file . We need to change database configurations and S3 bucket information. But since it is not secure to hardcode some critical data in side the code we use SSM parameters.So use the new "settings.py" file located in `solution folder` which contains boto3 function to retrieve database parameters.

- Movie and picture files are kept on S3 bucket named `awscapstones<name>blog` as object. You should create an S3 bucket and write name of it on `/src/cblog/settings.py` file as `AWS_STORAGE_BUCKET_NAME` variable. In addition, you must assign region of S3 as `AWS_S3_REGION_NAME` variable

- As for database; Users credentials and blog contents are going to be kept on RDS database. To connect EC2 to RDS, following variables must be assigned on `/src/cblog/settings.py` file after you create RDS;
    a. Database name - clarusway
    b. Database endpoint - "HOST" variables
    c. Port - 3306
    d. User -  >>>>> from parameter store
    e. PASSWORD >>>> from parameter store

-  So you need to  to change `/<youname>/capstone/username`a and  `/<youname>/capstone/password` with your own parameter name. And also `Database endpoint`

- before push check the requirement.txt . Pillow must be 8.4

===> PUSH YOUR CHANGES to github:
- git add .
- git commit -m "configuration changes"
- git push

```

## Step 8: Create certification for secure connection
We will use this later, but it may take a while to activate so we will create it now.

One of our requirements is to make our website connection in secure. It means, our web-site is going to use HTTPS protocol. To use this protocol, we need to get certification. Certification will encrypt data in transit between client and server. We can create our certificate for our DNS name with AWS certification Manager. Go to the certification manager console and click `request a certificate` button. Select `Request a public certificate`, then `request a certificate` ---> \*.clarusway.us ---> DNS validation ---> No tag ---> Review ---> click confirm and request button. Then it takes a while to be activated.

===> Make sure you click `Create DNS Records`

## Lunch break

## Step 9: Prepare a userdata to be utilized in Launch Template
userdata has already given by developer to us. In production environment. you should write it based on application and developer instructions. Lets go over it again.

First we will need a role for our ec2 instance:

- Go to IAM
- Click `Roles`
- Click `Create Role`
- Trusted Entity type: `AWS Service`
- Use case: `EC2`
- Add permission `AmazonS3FullAccess`
  `AmazonSSMFullAccess`
- Role name: aws-capstone-ec2-ssm-s3-full-access
- Click `Create role`

- Please check if this userdata is working or not. to do this create new instance in public subnet and show to students that it is working

Run one command at a time and look for any errors

<!-- BE CAREFULL!!!

After we create user data, and change the S3 and RDS variables into settings.py file, before we put it into the Launch Template, we should make sure if it is working. To do this, I'm gonna launch an instance with this userdata into the public subnet. After making sure that this works, I'll put it into the Launch Template -->

Instance properties:

- Name: aws-capstone-test-instance
- Ubuntu 22.04
- Key Pair: your key
- Subnet: aws-capstone-public-1a
- Security Group
  - aws-capstone-ec2-sg
  - aws-capstone-alb-sg
  - aws-capstone-nat-sg
- Advanced details:
  - IAM Instance Profile:
    - aws-capstone-ec2-ssm-s3-full-access

```bash
sudo su
apt-get update -y
apt-get install git -y
apt-get install python3 -y
cd /home/ubuntu/
TOKEN="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"
git clone https://$TOKEN@github.com/altazbhanji/capstone.git
cd /home/ubuntu/capstone
apt install python3-pip -y
apt-get install python3.10-dev default-libmysqlclient-dev -y
pip3 install -r requirements.txt
cd /home/ubuntu/capstone/src
python3 manage.py collectstatic --noinput
python3 manage.py makemigrations
python3 manage.py migrate
python3 manage.py runserver 0.0.0.0:80
```

In a browser check: - http://<public_dns_of_test_server>

If everything works, terminate the test server otherwise repeat or fix the steps

## Break

## Step 10: Create Launch Template

Ok, now we've create our VPC, defined security groups and written our user data and prepared our Github repo, launched NAT instance. Now we are going to create our Launch Template, and then we'll use it while creating our auto scaling group. Since our application which will be built on EC2 have to save blog's videos and pictures on S3, the first thing we will use an IAM role which allows EC2 to talk S3 Buckets.

To create Launch Template, go to the EC2 console and select `Launch Template` on the left hand menu. Click the Create Launch Template button.

```text
Launch template name                : aws-capstone-launch-template
Template version description        : Blog Web Page version 1
Amazon machine image (AMI)          : Ubuntu 22.04
Instance Type                       : t2.micro
Key Pair                            : your key pair
Subnet                              : Don't include
Security Groups                     : aws-capstone-ec2-sg
Storage (Volumes)                   : keep it as is
Resource tags                       : Key: Name   Value: aws-capstone-web-server
Advance Details:
    - IAM instance profile          : aws-capstone-ec2-ssm-s3-full-access
    - User Data
```

```bash
#!/bin/bash
# Insert your user data from above which you already verified
# comment out these two lines:
#       python3 manage.py makemigrations
#       python3 manage.py migrate
# since you don't have to run the migrations statements more than
# once since it re-configures the database
```

- create launch template

## Step 11: Create ALB and Target Group

Now, we'll create ALB and Target Group. Go to the EC2 console and click the Load Balancers on the left hand menu. click `create Load Balancer` button and select Application Load Balancer

```text
Application Load Balancer ---> Create
Basic Configuration:
Name                    : aws-capstone-alb
Schema                  : internet-facing
IP Address Type         : IPv4

Network mapping:
VPC                     : aws-capstone-vpc
Mappings                :
    - Availability zones:
        1. aws-capstone-public-subnet-1a
        2. aws-capstone-public-subnet-1b
Security Groups         :
Security Groups         : aws-capstone-alb-sg

Listeners and Routing   :
Listener HTTPS:443
Protocol HTTP ---> Port 443 ---> Default Action Create Target Group (New Window pops up)
    - Target Type         : Instances
    - Name                : aws-capstone-tg
    - Protocol            : HTTP
    - Port                : 80
    - Protocol version    : HTTP1
    - Health Check        :
      - Protocol          : HTTP
      - Path              : /
      - Port              : traffic port
      - Healthy threshold : 5
      - Unhealthy threshold : 2
      - Timeout           : 5
      - Interval          : 30
      - Success Code      : 200
click Next

Register Targets -->""We're not gonna add any EC2 here. While we create autoscaling group, it will ask us to show target group and ELB, then we'll indicate this target group there, so whenever autoscaling group launches new machine, it will registered this target group automatically.""
without register any target click Next: Review

then create "Target Group"

switch back to the ALB listener page and select newly created Target Group on the list

Click Add listener ---> Protocol HTTP ---> Port 80 ---> Default Action ---> Select newly created target group

Secure Listener Settings        :
    Security policy: ELBSecurityPolicy-2016-08
    Default ACM    : *.clarusway.us

```

- click `Create`

After creation of ALB, our ALB have to redirect http traffic to https port. Because our requirement wants to secure traffic. Thats why we should change listener rules. Go to the ALB console and select Listeners sub-section

```text
select HTTP: 80 rule ---> Select Manage Rules | Edit rules
- Check Default under "Listener Rules"
- Click `Actions | Edit rule`
- Choose Redirect to URL
- Enter 443 for Port number
- Status code is "301 - permanently moved"
```

Test to see that your load balancer is working:

- Type http://<loadbalancer.dns>
  - It should re-direct you to https and tell you that the connection is not secure
- Type https://<loadbalancer.dns>
  - It should tell you connection is not secure

In any case, if you proceed past the browser warnings, you should see a 5xx error. Why?
(EC2 instances not connected yet)

## Step 12: Create Autoscaling Group with Launch Template

Now we'll create our Autoscaling Group. We'll use newly created Launch Template. Go to the Autoscaling Group on the left hand side menu. Click create Autoscaling group.

- Choose launch template or configuration

```text
Auto Scaling group name         : aws-capstone-asg
Launch Template                 : aws-capstone-launch-template
```

- Configure settings

```text
Network                         :
    - VPC                       : aws-capstone-vpc
    - Subnets                   : Private 1a and Private 1b
```

Configure advanced options

```text
- Load balancing                                : Attach to an existing load balancer
- Choose from your load balancer target groups  : awscapstoneTargetGroup
- Health Checks
    - Health Check Type             : ELB
    - Health check grace period     : 300
```

- Configure group size and scaling policies

```text
Group size
    - Desired capacity  : 2
    - Minimum capacity  : 2
    - Maximum capacity  : 4
Scaling policies
    - Target tracking scaling policy
        - Scaling policy name       : Target Tracking Policy
        - Metric Type               : Average CPU utilization
        - Target value              : 70
```

<!-- WARNING!!! Sometimes your EC2 has a problem after you create autoscaling group, If you need to look inside one of your instance to make sure where the problem is, please follow these steps...

```bash
eval $(ssh-agent) (your local)
ssh-add xxxxxxxxxx.pem   (your local )
ssh -A ec2-user@<Public IP or DNS name of NAT instance> (your local)
ssh ubuntu@<Private IP of web server>  (in NAT instance)
You are in the private EC2 instance
``` -->

````

## break

## Step 13: Create Cloudfront in front of ALB
We'll define cloudfront in front of ALB. Lets go ahead and configure cloudfront. Go to the cloudfront menu and click start
- Origin Settings
```text
Origin Domain Name          : aws-capstone-ALB-1947210493.us-east-2.elb.amazonaws.com
Protocol                    : Match Viewer
HTTP Port                   : 80
HTTPS                       : 443
Minimum Origin SSL Protocol : Keep it as is
Path                 : Leave empty (this means, define for root '/')
Name                        : aws-capstone-alb-origin
Add custom header           : No header
Enable Origin Shield        : No
Additional settings         : Keep it as is
````

Default Cache Behavior Settings

```text
Path pattern                                : Default (*)
Compress objects automatically              : Yes
Viewer Protocol Policy                      : Redirect HTTP to HTTPS
Allowed HTTP Methods                        : GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE
Cached HTTP Methods                         : Select OPTIONS
Cache key and origin requests
- Use legacy cache settings
  Headers     : Include the following headers
    Add Header
    - Accept
    - Accept-Charset
    - Accept-Language
    - Accept-Datetime
    - Accept-Encoding
    - Authorization
    - Host
    - Origin
    - Referrer
    - Cloudfront-Forwarded-Proto
Forward Cookies                         : All
Query String Forwarding and Caching     : All
Other stuff                             : Keep them as are
```

- Function Associations : Leave as is
- WAF : Do not enable

- Distribution Settings

```text
Price Class                             : Use NA &Europe
Alternate Domain Names                  : www.clarusway.us
SSL Certificate                         : Custom SSL Certificate (example.com) ---> Select your certificate created before
Other stuff                             : Keep them as is
```

Click `Create distribution`

==> Now test: - Check your LB DNS URL (with http and https)

## Step 14: Create Route 53 with Failover settings

Come to the Route53 console and select Health checks on the left hand menu. Click create health check

Configure health check

```text
Name                : aws-capstone-health-check
What to monitor     : Endpoint
Specify endpoint by : Domain Name
Protocol            : HTTP
Domain Name         : Write cloudfront domain name
Port                : 80
Path                : leave it blank
Other stuff         : Keep them as are
```

- Click Hosted zones on the left hand menu

- click your Hosted zone : clarusway.us

- Create Failover scenario

- Click Create Record

- Select Failover ---> Click Next

```text
Configure records
Record name             : www.clarusway.us
Record Type             : A - Routes traffic to an IPv4 address and some AWS resources
TTL                     : 300

First we'll create a primary record for cloudfront

Failover records to add to clarusway.us ---> Define failover record

Value/Route traffic to  : Alias to cloudfront distribution
                          - Select created cloudfront DNS
Failover record type    : Primary
Health check            : aws-capstone-health-check
Record ID               : Cloudfront as Primary Record
----------------------------------------------------------------
Second we'll create secondary record for S3

Failover records to add to clarusway.us ---> Define failover record

Value/Route traffic to  : Alias to S3 website endpoint
                          - Select Region
                          - Your created bucket name emerges ---> Select it
Failover record type    : Secondary
Health check            : No health check
Record ID               : S3 Bucket for Secondary record type
```

- click create records

==> Now test: - Your CloudFront URL (with http and https) - www.<your_domain> (with http and https)

# break

Ok we've completed our website's configurations. Now, we'll create our DynamoDB table and Lambda function.

## Step 15: Create DynamoDB Table
Go to the Dynamo Db table and click create table button

- Create DynamoDB table

```text
Name            : aws-capstone-dynamo
Primary key     : id
Other Stuff     : Keep them as are
click create
```

## Step 16-17: Create Lambda function

Now we'll create a lambda function with the python code given by developer team. Before we create our Lambda function, we should create IAM role that we'll use for Lambda function. Go to the IAM console and select role on the left hand menu, then create role button

```text
Select Lambda as trusted entity ---> click Next:Permission
Choose: - AmazonS3fullaccess
        - AmazonDynamoDBFullAccess
        - AWSLambdaBasicExecutionRole
No tags
Role Name           : aws-capstone-lambda-role
Role description    : This role give a permission to lambda to reach S3 and DynamoDB
```

then, go to the Lambda Console and click create function

- Basic Information

```text

Function Name           : aws-capstone-lambda-function
Runtime                 : Python 3.10
Permissions
Use an existing role    : aws-capstone-lambda-role
```

Click `Create function`

- Modify Lambda Function timeout to be 30 seconds

- Now we'll go to the S3 bucket belongs our website and create an event to trigger our Lambda function.

## Step 18-19: Create trigger for Lambda Function

- Go to the `aws-capstone-lambda-function` lambda Function
- Click `Add trigger`
- Select `S3` as Source
- Select `awscapstones<name>blog` bucket
- Choose `All object create events`
- Click the `I acknowledge` check box
- Click `Add` to add the trigger

````
- Go to the code part and select lambda_function.py ---> remove default code and paste a code on below. If you give DynamoDB a different name, please make sure to change it into the code.

```python
import json
import boto3

def lambda_handler(event, context):
    s3 = boto3.client("s3")

    if event:
        print("Event: ", event)
        filename = str(event['Records'][0]['s3']['object']['key'])
        timestamp = str(event['Records'][0]['eventTime'])
        event_name = str(event['Records'][0]['eventName']).split(':')[0][6:]

        filename1 = filename.split('/')
        filename2 = filename1[-1]

        dynamo_db = boto3.resource('dynamodb')
        dynamoTable = dynamo_db.Table('aws-capstone-dynamo')

        dynamoTable.put_item(Item = {
            'id': filename2,
            'timestamp': timestamp,
            'Event': event_name,
        })

    return "Lambda success"
````

- Click deploy and all set. go to the website and add a new post with photo, then control if their record is written on DynamoDB.

- WE ALL SET

## Clean-up

1. CloudFront>>>>>Disable>>>>>>Delete
2. RDS
3. RDS Subnet Group (After RDS Deleted)
4. Lambda Function
5. DynamoDB
6. R53 healthcheck
7. R53 failover records
8. S3 buckets (first objects)
9. IAM roles
10. NAT instance >>>>>terminate
11. ALB
12. TG
13. LT
14. ASG
15. Endpoint
16. internet gateway>>>>>detach>>>>>delete
17. subnets
18. private RT
19. VPC
20. Certificates
21. GitHub tokens
