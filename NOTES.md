# 14 - Automation with Python

## 1 - Introduction to Boto Library (AWS SDK for Python)

Boto is the Amazon Web Services (AWS) Software Development Kit (SDK) for Python, which allows Python developers to write software that makes use of AWS services.

## 2 - Install Boto3 and connect to AWS

Install Boto3 using pip:

```bash
pip install boto3
```

## 3 - Getting familiar with Boto

Boto documentation: https://boto3.amazonaws.com/v1/documentation/api/latest/index.html or https://docs.aws.amazon.com/boto3/latest/

```py
import boto3

# Create an ec2 client
ec2 = boto3.client('ec2', region_name='us-west-2')

# create an ec2 resource
ec2_resource = boto3.resource('ec2', region_name='us-west-2')

new_vpc = ec2_resource.create_vpc(CidrBlock='10.0.0.0/16')

new_vpc.create_tags(Tags=[{"Key": "Name", "Value": "my-vpc"}])

new_vpc.create_subnet(CidrBlock='10.0.1.0/24')

new_vpc.create_subnet(CidrBlock='10.0.2.0/24')

# List all vpcs
response = ec2.describe_vpcs()
for vpc in response['Vpcs']:
    print(vpc['VpcId'])
```

Boto client vs resource:

- Client: 
  - Low-level service access, maps directly to AWS service APIs.
- Resource: 
  - Higher-level, object-oriented API, abstracts some of the low-level details.
  - Return resource objects that can be used for subsequent calls.

## 4 - Terraform vs Python - understand when to use which tool

Terraform is a declarative infrastructure as code tool, while Python (with Boto3) is an imperative programming language.
Terraform is idempotent & best suited for provisioning and managing infrastructure in a predictable and repeatable manner, while Python is better for automating tasks, integrating with other systems, and performing complex logic.

## 5 - Health Check: EC2 Status Checks

Create 3 instances with terraform

```tf
provider "aws" {
  region = "eu-west-3"
}

variable vpc_cidr_block {}
variable subnet_cidr_block {}
variable avail_zone {}
variable env_prefix {}
variable instance_type {}
variable my_ip {}
variable public_key_location {}

data "aws_ami" "amazon-linux-image" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

output "ami_id" {
  value = data.aws_ami.amazon-linux-image.id
}

resource "aws_vpc" "myapp-vpc" {
  cidr_block = var.vpc_cidr_block
  tags = {
      Name = "${var.env_prefix}-vpc"
  }
}

resource "aws_subnet" "myapp-subnet-1" {
  vpc_id = aws_vpc.myapp-vpc.id
  cidr_block = var.subnet_cidr_block
  availability_zone = var.avail_zone
  tags = {
      Name = "${var.env_prefix}-subnet-1"
  }
}

resource "aws_security_group" "myapp-sg" {
  name   = "myapp-sg"
  vpc_id = aws_vpc.myapp-vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.my_ip]
  }

  ingress {
    from_port   = 8080
    to_port     = 8080
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    cidr_blocks     = ["0.0.0.0/0"]
    prefix_list_ids = []
  }

  tags = {
    Name = "${var.env_prefix}-sg"
  }
}

resource "aws_internet_gateway" "myapp-igw" {
	vpc_id = aws_vpc.myapp-vpc.id
    
    tags = {
     Name = "${var.env_prefix}-internet-gateway"
   }
}

resource "aws_route_table" "myapp-route-table" {
   vpc_id = aws_vpc.myapp-vpc.id

   route {
     cidr_block = "0.0.0.0/0"
     gateway_id = aws_internet_gateway.myapp-igw.id
   }

   # default route, mapping VPC CIDR block to "local", created implicitly and cannot be specified.

   tags = {
     Name = "${var.env_prefix}-route-table"
   }
 }

# Associate subnet with Route Table
resource "aws_route_table_association" "a-rtb-subnet" {
  subnet_id      = aws_subnet.myapp-subnet-1.id
  route_table_id = aws_route_table.myapp-route-table.id
}

resource "aws_key_pair" "ssh-key" {
  key_name   = "myapp-key"
  public_key = file(var.public_key_location)
}

output "server-ip" {
    value = aws_instance.myapp-server.public_ip
}

resource "aws_instance" "myapp-server" {
  ami                         = data.aws_ami.amazon-linux-image.id
  instance_type               = var.instance_type
  key_name                    = "myapp-key"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.myapp-subnet-1.id
  vpc_security_group_ids      = [aws_security_group.myapp-sg.id]
  availability_zone			      = var.avail_zone

  tags = {
    Name = "${var.env_prefix}-server"
  }

  user_data = file("entry-script.sh")
  
  user_data_replace_on_change = true

}

resource "aws_instance" "myapp-server-two" {
  ami                         = data.aws_ami.amazon-linux-image.id
  instance_type               = var.instance_type
  key_name                    = "myapp-key"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.myapp-subnet-1.id
  vpc_security_group_ids      = [aws_security_group.myapp-sg.id]
  availability_zone			      = var.avail_zone

  tags = {
    Name = "${var.env_prefix}-server-two"
  }

  user_data = file("entry-script.sh")
  
  user_data_replace_on_change = true

}

resource "aws_instance" "myapp-server-three" {
  ami                         = data.aws_ami.amazon-linux-image.id
  instance_type               = var.instance_type
  key_name                    = "myapp-key"
  associate_public_ip_address = true
  subnet_id                   = aws_subnet.myapp-subnet-1.id
  vpc_security_group_ids      = [aws_security_group.myapp-sg.id]
  availability_zone			      = var.avail_zone

  tags = {
    Name = "${var.env_prefix}-server-three"
  }

  user_data = file("entry-script.sh")
  
  user_data_replace_on_change = true

}
```

Check the instance status with python

```py
import boto3
import schedule

ec2_client = boto3.client('ec2', region_name="eu-west-3")
ec2_resource = boto3.resource('ec2', region_name="eu-west-3")


def check_instance_status():
    statuses = ec2_client.describe_instance_status(
        IncludeAllInstances=True
    )
    for status in statuses['InstanceStatuses']:
        ins_status = status['InstanceStatus']['Status']
        sys_status = status['SystemStatus']['Status']
        state = status['InstanceState']['Name']
        print(f"Instance {status['InstanceId']} is {state} with instance status {ins_status} and system status {sys_status}")
    print("#############################\n")


schedule.every(5).minutes.do(check_instance_status)

while True:
    schedule.run_pending()
```

##  6 - Write a Scheduled Task in Python

pip install schedule

```bash
pip install schedule
```

## 7 - Configure Server: Add Environment Tags to EC2 Instances

If organization has AWS resources in different regions, we can use tags to identify the environment of each resource. For example, we can add a tag called "Environment" with values like "Development", "Staging", or "Production" to each EC2 instance.

Python boto3 code to add tags to EC2 instances:

```py
import boto3

ec2_client_paris = boto3.client('ec2', region_name="eu-west-3")
ec2_resource_paris = boto3.resource('ec2', region_name="eu-west-3")
ec2_client_frankfurt = boto3.client('ec2', region_name="eu-central-1")
ec2_resource_frankfurt = boto3.resource('ec2', region_name="eu-central-1")

instance_ids_paris = []
instance_ids_frankfurt = []

# fetch instances & iterate to get their ids
reservations_paris = ec2_client_paris.describe_instances()

for reservation in reservations_paris['Reservations']:
  for instance in reservation['Instances']:
    instance_ids_paris.append(instance['InstanceId'])

# add tags to instances
ec2_resource_paris.create_tags(
  Resources=instance_ids_paris,
  Tags=[
    {
      'Key': 'Environment',
      'Value': 'Development'
    }
  ]
)

# fetch instances & iterate to get their ids
reservations_frankfurt = ec2_client_frankfurt.describe_instances()

for reservation in reservations_frankfurt['Reservations']:
  for instance in reservation['Instances']:
    instance_ids_frankfurt.append(instance['InstanceId'])

# add tags to instances
ec2_resource_frankfurt.create_tags(
  Resources=instance_ids_frankfurt,
  Tags=[
    {
      'Key': 'Environment',
      'Value': 'Production'
    }
  ]
)
```

## 8 - EKS cluster information

Boto3 can be used to fetch information about an EKS cluster, such as its name, status, endpoint, and version. Here's an example code snippet:

```py
import boto3
eks_client = boto3.client('eks', region_name="eu-west-3")

clusters = eks_client.list_clusters()['clusters']

for cluster in clusters:
  response = eks_client.describe_cluster(name=cluster)
  cluster_info = response['cluster']

  print(f"Cluster Name: {cluster_info['name']}")
  print(f"Status: {cluster_info['status']}")
  print(f"Endpoint: {cluster_info['endpoint']}")
  print(f"Version: {cluster_info['version']}")
```

## 9 - Backup EC2 Volumes: Automate creating Snapshots

Create 2 EC2 instances on AWS console & add "Name" tags "dev" & "prod" respectively. Run the following Python script to create snapshots of EC2 volumes with the "Name" tag "prod". The script uses the Boto3 library to interact with the AWS API and the schedule library to run the snapshot creation task every day.

Tag the created volumes with "Name" tag "prod" to identify them for snapshot creation. The script fetches all volumes with the "Name" tag "prod" and creates snapshots for each of them. The snapshots are created in the same region as the volumes.

EC2 volumes can be backed up by creating snapshots. Here's an example code snippet to automate creating snapshots of EC2 volumes:

```py
import boto3
import schedule

ec2_client = boto3.client('ec2', region_name="eu-central-1")

def create_volume_snapshots():
  volumes = ec2_client.describe_volumes(
    Filters=[
      {
        'Name': 'tag:Name',
        'Values': ['prod']
      }
    ]
  )

  for volume in volumes['Volumes']:
    new_snapshot = ec2_client.create_snapshot(
      VolumeId=volume['VolumeId']
    )
    print(new_snapshot)

schedule.every().day.do(create_volume_snapshots)

while True:
    schedule.run_pending()
```

## 10 - Automate cleanup of old Snapshots

To reduce the number of snapshots and save costs, we can automate the cleanup of old snapshots. Here's an example code snippet to delete snapshots and only keep the 2 most recent snapshots for each volume:

```py
import boto3
from operator import itemgetter

ec2_client = boto3.client('ec2', region_name="eu-central-1")

volumes = ec2_client.describe_volumes(
  Filters=[
    {
      'Name': 'tag:Name',
      'Values': ['prod']
    }
  ]
)

for volume in volumes['Volumes']:
  snapshots = ec2_client.describe_snapshots(
    OwnerIds=['self'],
    Filters=[
      {
        'Name': 'volume-id',
        'Values': [volume['VolumeId']]
      }
    ]
  )

  sorted_by_date = sorted(snapshots['Snapshots'], key=itemgetter('StartTime'), reverse=True)

  for snap in sorted_by_date[2:]:
    response = ec2_client.delete_snapshot(
      SnapshotId=snap['SnapshotId']
    )
    print(response)
```

## 11 - Automate restoring EC2 Volume from the Backup

Snapshots can be used to restore EC2 volumes. Here's an example code snippet to automate restoring an EC2 volume from a snapshot & attaching it to an EC2 instance:

```py
import boto3
from operator import itemgetter

ec2_client = boto3.client('ec2', region_name="eu-central-1")
ec2_resource = boto3.resource('ec2', region_name="eu-central-1")

instance_id = "i-0671d0fe02906a969"

volumes = ec2_client.describe_volumes(
  Filters=[
    {
      'Name': 'attachment.instance-id',
      'Values': [instance_id]
    }
  ]
)

instance_volume = volumes['Volumes'][0]

snapshots = ec2_client.describe_snapshots(
  OwnerIds=['self'],
  Filters=[
    {
      'Name': 'volume-id',
      'Values': [instance_volume['VolumeId']]
    }
  ]
)

latest_snapshot = sorted(snapshots['Snapshots'], key=itemgetter('StartTime'), reverse=True)[0]
print(latest_snapshot['StartTime'])

new_volume = ec2_client.create_volume(
  SnapshotId=latest_snapshot['SnapshotId'],
  AvailabilityZone="eu-central-1a",
  TagSpecifications=[
    {
      'ResourceType': 'volume',
      'Tags': [
        {
          'Key': 'Name',
          'Value': 'prod'
      }
      ]
    }
  ]
)

while True:
  vol = ec2_resource.Volume(new_volume['VolumeId'])
  print(vol.state)
  if vol.state == 'available':
    ec2_resource.Instance(instance_id).attach_volume(
      VolumeId=new_volume['VolumeId'],
      Device='/dev/xvdb'
    )
    break
```

## 12 - Handling Errors

To handle errors in Boto3, we can use try-except blocks to catch exceptions and handle them gracefully.

## 13 - Website Monitoring 1: Scheduled Task to Monitor Application Health

Create a server on linode and install docker.

```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Run nginx container on the server.

```bash
sudo docker run -d -p 8080:80 --name nginx-server nginx
```

Install "requests" library in python to make http requests.

```bash
pip install requests
```

Send request to check the health of the application running on the server.

```py
import requests

def check_website_health():
    url = "http://<server-ip>:8080"
    try:
        response = requests.get(url)
        if response.status_code == 200:
            print(f"Website is up and running. Status code: {response.status_code}")
        else:
            print(f"Website is down. Status code: {response.status_code}")
    except requests.exceptions.RequestException as e:
        print(f"An error occurred: {e}")
```

## 14 - Website Monitoring 2: Automated Email Notification

Setup gmail as email server to send email notifications using python smtplib library.

Navigate to https://myaccount.google.com/lesssecureapps and enable "Allow less secure apps" to allow sending emails from python script. If you have 2FA enabled, you need to create an app password and use it instead of your regular password in the url https://myaccount.google.com/apppasswords.

Use "os" library to read the email and password from environment variables instead of hardcoding them in the script.

```py
import smtplib
import os

EMAIL_ADDRESS = os.environ.get('EMAIL_ADDRESS')
EMAIL_PASSWORD = os.environ.get('EMAIL_PASSWORD')

def send_notification(email_msg):
    print('Sending an email...')
    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.ehlo()
        smtp.login(EMAIL_ADDRESS, EMAIL_PASSWORD)
        message = f"Subject: SITE DOWN\n{email_msg}"
        smtp.sendmail(EMAIL_ADDRESS, EMAIL_ADDRESS, message)
```

## 15 - Website Monitoring 3: Restart Application and Reboot Server

zxvb dgei wivi umif

USe "paramako" library to connect to the server via ssh and restart the application or reboot the server.

Install "linode_api4" library to connect to the linode server and reboot it.

```py
import requests
import smtplib
import os
import paramiko
import linode_api4
import time
import schedule
import dotenv

dotenv.load_dotenv()

EMAIL_ADDRESS = os.environ.get('EMAIL_ADDRESS')
EMAIL_PASSWORD = os.environ.get('EMAIL_PASSWORD')
LINODE_TOKEN = os.environ.get('LINODE_TOKEN')
KEY_FILENAME = os.environ.get('KEY_FILENAME')


def restart_server_and_container():
    # restart linode server
    print('Rebooting the server...')
    client = linode_api4.LinodeClient(LINODE_TOKEN)
    nginx_server = client.load(linode_api4.Instance, 106818789)
    nginx_server.reboot()

    # restart the application
    while True:
        nginx_server = client.load(linode_api4.Instance, 106818789)
        if nginx_server.status == 'running':
            time.sleep(5)
            restart_container()
            break


def send_notification(email_msg):
    print('Sending an email...')
    with smtplib.SMTP('smtp.gmail.com', 587) as smtp:
        smtp.starttls()
        smtp.ehlo()
        smtp.login(EMAIL_ADDRESS, EMAIL_PASSWORD)
        message = f"Subject: SITE DOWN\n{email_msg}"
        smtp.sendmail(EMAIL_ADDRESS, EMAIL_ADDRESS, message)


def restart_container():
    print('Restarting the application...')
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh.connect(hostname='172.235.6.68', username='root', key_filename=KEY_FILENAME)
    stdin, stdout, stderr = ssh.exec_command('docker start 6c2e0f336181')
    print(stdout.readlines())
    ssh.close()


def monitor_application():
    try:
        response = requests.get('http://172-235-6-68.ip.linodeusercontent.com:8080/')
        if response.status_code == 200:
            print('Application is running successfully!')
        else:
            print('Application Down. Fix it!')
            msg = f'Application returned {response.status_code}'
            send_notification(msg)
            restart_container()
    except Exception as ex:
        print(f'Connection error happened: {ex}')
        msg = 'Application not accessible at all'
        send_notification(msg)
        restart_server_and_container()


schedule.every(5).seconds.do(monitor_application)

while True:
    schedule.run_pending()
```
