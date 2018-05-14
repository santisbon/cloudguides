   * [Prerequisites](#prerequisites)
   * [EC2 Instance types](#ec2-instance-types)
      * [General Purpose](#general-purpose)
      * [Compute Optimized](#compute-optimized)
      * [Memory Optimized](#memory-optimized)
      * [Accelerated Computing](#accelerated-computing)
      * [Storage Optimized](#storage-optimized)
   * [EBS Volume Types](#ebs-volume-types)
      * [Solid-State Drives (SSD)](#solid-state-drives-ssd)
      * [Hard Disk Drives (HDD)](#hard-disk-drives-hdd)
   * [Configuration parameters](#configuration-parameters)
   * [Virtual Private Cloud (VPC)](#virtual-private-cloud-vpc)
   * [Subnets](#subnets)
   * [Network ACL for web servers](#network-acl-for-web-servers)
   * [Internet Gateway](#internet-gateway)
   * [Internet Route Table](#internet-route-table)
   * [EC2 security groups on the VPC](#ec2-security-groups-on-the-vpc)
   * [RDS](#rds)
      * [RDS Aurora](#rds-aurora)
      * [RDS MySQL/MariaDB/PostgreSQL/Oracle/SQL Server](#rds-mysqlmariadbpostgresqloraclesql-server)
   * [DynamoDB](#dynamodb)
      * [Run it locally](#run-it-locally)
   * [Key pairs](#key-pairs)
   * [Finding images](#finding-images)
   * [Create instances.](#create-instances)
   * [Network Address Translation (NAT)](#network-address-translation-nat)
      * [NAT instance](#nat-instance)
      * [NAT Gateway](#nat-gateway)
   * [CloudFormation](#cloudformation)

# Prerequisites

Have an IAM policy with the needed permissions (e.g. AdministratorAccess has all permissions).  
If running the aws cli from an EC2 instance use a role with that policy.  
If using a user with access keys from your workstation, set access key ID and secret access key.

Use the "aws configure" command to:
1. Set the output format to json.
2. Set the default region e.g. us-east-1.
3. If using access keys, set the access key id and secret access key.

These settings will be stored in ~/.aws/config and ~/.aws/credentials.

# EC2 Instance types

D = Density  
R = RAM  
M = Main choice  
C = Compute  
G = Graphics-intensive  
I = IOPS  
F = Field Programmable Array (FPGA)  
T = Cheap  
P = Graphics (GPU)  
X = Extreme Memory  

## General Purpose
T2 - EBS-only storage. Burstable performance.  
M4 - EBS-only storage. EBS-optimized. Enhanced Networking.  
M3 - SSD-based instance storage.  

## Compute Optimized
C4 - EBS-only storage. EBS-optimized. Enhanced Networking. Clustering.  
C3 - SSD-based instance storage. Enhanced Networking. Clustering.  

## Memory Optimized
X1 - Up to 1,952 GiB DDR4 instance memory. SSD storage. EBS-optimized.  
R4 - DDR4 memory. Enhanced Networking.  
R3 - SSD storage. Enhanced Networking.  

## Accelerated Computing
P2 - General Purpose GPU compute. Enhanced Networking. EBS-optimized.  
G2 - For graphics-intensive apps.  
F1 - Customizable hardware acceleration with FPGAs.  

## Storage Optimized
I3 - High I/O. Enhanced Networking.  
D2 - Dense storage. HDD storage. High disk throughput. Enhanced Networking.  

# EBS Volume Types

## Solid-State Drives (SSD)
gp2 - General purpose SSD. Up to 10,000 IOPS.  
io1 - Provisioned IOPS. Good for > 10,000 IOPS. Up to 20,000 IOPS per volume.  

## Hard Disk Drives (HDD)
st1 - Throughput Optimized HDD. Good for sequential data. datawarehousing.  
sc1 - Cold HDD. Lowest cost. Good for infrequently accessed volumes.  
standard (previous gen cold) - Only magnetic type that can be boot volume.  

# Configuration parameters
```bash
# Amazon Machine Image ID for Amazon Linux
readonly AMZN_LINUX_AMI_ID='ami-0b33d91d'
readonly AMZN_LINUX_NAT_AMI_ID='ami-184dc970'

# We'll use two availability zones.
# Note: Not all services are available in all regions and availability zones.
az_1='us-east-1b'
az_2='us-east-1c'

# CIDR blocks for VPC and subnets
vpc_cidr='10.0.0.0/16'

web_cidr_1='10.0.1.0/24'
web_cidr_2='10.0.2.0/24'

app_cidr_1='10.0.3.0/24'
app_cidr_2='10.0.4.0/24'

db_cidr_1='10.0.5.0/24'
db_cidr_2='10.0.6.0/24'

all_cidr='0.0.0.0/0'

# CIDR block for my external ip address
my_cidr="$(dig +short myip.opendns.com @resolver1.opendns.com)/32"

# Ports
http_port=80
https_port=443
mysql_port=3306

# RDS master credentials
rds_master_user='masteruser'
rds_master_password='masterpassword'
# DB Instance types
# Note: Not all instance classes are available in all regions for all DB engines
rds_aurora_inst_type='db.t2.medium'
rds_inst_type='db.t2.micro'
# Allocated storage for RDS
rds_alloc_storage=5

# Server instance types
app_inst_type='t2.micro'
web_inst_type='t2.micro'
```

# Virtual Private Cloud (VPC)
Create a VPC and tag it. This also creates a:
- Main route table (local traffic)
- Default security group (inbound from group, outbound everywhere)
- Default network acl (allow all traffic inbound and outbound)
```bash
my_vpc_id="$(aws ec2 create-vpc \
  --cidr-block ${vpc_cidr} \
  --query 'Vpc.VpcId' | tr -d  '"')"

aws ec2 create-tags --resources ${my_vpc_id} --tags Key=Name,Value='My VPC'

# aws ec2 describe-vpc-attribute \
#   --vpc-id ${my_vpc_id} --attribute enableDnsSupport
# aws ec2 describe-vpc-attribute \
#   --vpc-id ${my_vpc_id} --attribute enableDnsHostnames

# --enable-dns-hostnames | --no-enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id ${my_vpc_id} --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id ${my_vpc_id} --enable-dns-hostnames
```

# Subnets
Create two subnets in different AZs for web servers and tag them.
```bash
web_subnet_id_1="$(aws ec2 create-subnet \
  --vpc-id ${my_vpc_id} \
  --cidr-block ${web_cidr_1} \
  --availability-zone ${az_1} \
  --query 'Subnet.SubnetId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${web_subnet_id_1} \
  --tags Key=Name,Value="${web_cidr_1} - ${az_1}"

web_subnet_id_2="$(aws ec2 create-subnet \
  --vpc-id ${my_vpc_id} \
  --cidr-block ${web_cidr_2} \
  --availability-zone ${az_2} \
  --query 'Subnet.SubnetId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${web_subnet_id_2} \
  --tags Key=Name,Value="${web_cidr_2} - ${az_2}"

# Create two subnets in different AZs for app servers and tag them
app_subnet_id_1="$(aws ec2 create-subnet \
  --vpc-id ${my_vpc_id} \
  --cidr-block ${app_cidr_1} \
  --availability-zone ${az_1} \
  --query 'Subnet.SubnetId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${app_subnet_id_1} \
  --tags Key=Name,Value="${app_cidr_1} - ${az_1}"

app_subnet_id_2="$(aws ec2 create-subnet \
  --vpc-id ${my_vpc_id} \
  --cidr-block ${app_cidr_2} \
  --availability-zone ${az_2} \
  --query 'Subnet.SubnetId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${app_subnet_id_2} \
  --tags Key=Name,Value="${app_cidr_2} - ${az_2}"

# Create two subnets in different AZs for RDS instances and tag them
db_subnet_id_1="$(aws ec2 create-subnet \
  --vpc-id ${my_vpc_id} \
  --cidr-block ${db_cidr_1} \
  --availability-zone ${az_1} \
  --query 'Subnet.SubnetId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${db_subnet_id_1} \
  --tags Key=Name,Value="${db_cidr_1} - ${az_1}"

db_subnet_id_2="$(aws ec2 create-subnet \
  --vpc-id ${my_vpc_id} \
  --cidr-block ${db_cidr_2} \
  --availability-zone ${az_2} \
  --query 'Subnet.SubnetId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${db_subnet_id_2} \
  --tags Key=Name,Value="${db_cidr_2} - ${az_2}"
```

# Network ACL for web servers
```bash
web_acl_id="$(aws ec2 create-network-acl \
  --vpc-id ${my_vpc_id} \
  --query 'NetworkAcl.NetworkAclId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${web_acl_id} \
  --tags Key=Name,Value='ACL - Web'

# Inbound rules for htpp/s, ssh, and ephemeral ports

aws ec2 create-network-acl-entry \
  --network-acl-id ${web_acl_id} \
  --rule-number 100 \
  --protocol 'tcp' \
  --port-range "From=${http_port},To=${http_port}" \
  --cidr-block ${all_cidr}
  --rule-action 'allow' \
  --ingress

aws ec2 create-network-acl-entry \
  --network-acl-id ${web_acl_id} \
  --rule-number 200 \
  --protocol 'tcp' \
  --port-range "From=${https_port},To=${https_port}" \
  --cidr-block ${all_cidr}
  --rule-action 'allow' \
  --ingress

aws ec2 create-network-acl-entry \
  --network-acl-id ${web_acl_id} \
  --rule-number 300 \
  --protocol 'tcp' \
  --port-range 'From=22,To=22' \
  --cidr-block ${my_cidr}
  --rule-action 'allow' \
  --ingress

aws ec2 create-network-acl-entry \
  --network-acl-id ${web_acl_id} \
  --rule-number 400 \
  --protocol 'tcp' \
  --port-range 'From=1024,To=65535' \
  --cidr-block ${all_cidr}
  --rule-action 'allow' \
  --ingress

# Outbound rules for htpp/s, ssh, and ephemeral ports

aws ec2 create-network-acl-entry \
  --network-acl-id ${web_acl_id} \
  --rule-number 100 \
  --protocol 'tcp' \
  --port-range "From=${http_port},To=${http_port}" \
  --cidr-block ${all_cidr}
  --rule-action 'allow' \
  --egress

aws ec2 create-network-acl-entry \
  --network-acl-id ${web_acl_id} \
  --rule-number 200 \
  --protocol 'tcp' \
  --port-range "From=${https_port},To=${https_port}" \
  --cidr-block ${all_cidr}
  --rule-action 'allow' \
  --egress

aws ec2 create-network-acl-entry \
  --network-acl-id ${web_acl_id} \
  --rule-number 300 \
  --protocol 'tcp' \
  --port-range 'From=22,To=22' \
  --cidr-block ${my_cidr}
  --rule-action 'allow' \
  --egress

aws ec2 create-network-acl-entry \
  --network-acl-id ${web_acl_id} \
  --rule-number 400 \
  --protocol 'tcp' \
  --port-range 'From=1024,To=65535' \
  --cidr-block ${all_cidr}
  --rule-action 'allow' \
  --egress

# Change the association for both subnets from the default ACL to the new one

assoc_id="$(aws ec2 describe-network-acls \
  --query "NetworkAcls[?VpcId==\`${my_vpc_id}\`].Associations[] \
  | [?SubnetId==\`${web_subnet_id_1}\`].NetworkAclAssociationId" \
  | tr -d '][\"[:space:]')"

aws ec2 replace-network-acl-association \
  --association-id ${assoc_id}
  --network-acl-id ${web_acl_id}

assoc_id="$(aws ec2 describe-network-acls \
  --query "NetworkAcls[?VpcId==\`${my_vpc_id}\`].Associations[] \
  | [?SubnetId==\`${web_subnet_id_2}\`].NetworkAclAssociationId" \
  | tr -d '][\"[:space:]')"

aws ec2 replace-network-acl-association \
  --association-id ${assoc_id}
  --network-acl-id ${web_acl_id}

################################################################################
# TODO: Network ACL for app servers
# ephemeral ports only on inbound rules (return traffic from the internet)
################################################################################

################################################################################
# TODO: Network ACL for DB instances
# ephemeral ports only on inbound rules (return traffic from the internet)
################################################################################

################################################################################
# TODO: VPC Flow logs
# Needs: A role that can write to CloudWatch,
# a CloudWatch Log Group and Log Stream
################################################################################
```

# Internet Gateway
```bash
# Create an Internet gateway, tag it, and attach it to our VPC
my_igw_id="$(aws ec2 create-internet-gateway \
  --query 'InternetGateway.InternetGatewayId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${my_igw_id} \
  --tags Key=Name,Value='IGW - My VPC'

aws ec2 attach-internet-gateway \
  --internet-gateway-id ${my_igw_id} \
  --vpc-id ${my_vpc_id}
```

# Internet Route Table
```bash
# Create an Internet access route table for the VPC
internet_route_table_id="$(aws ec2 create-route-table \
  --vpc-id ${my_vpc_id} \
  --query 'RouteTable.RouteTableId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${internet_route_table_id} \
  --tags Key=Name,Value='RTB - Internet route table'

# Add routes to the route table
aws ec2 create-route \
  --route-table-id ${internet_route_table_id} \
  --destination-cidr-block ${all_cidr} \
  --gateway-id ${my_igw_id}

# Associate the route table with the subnets we want
aws ec2 associate-route-table \
  --route-table-id ${internet_route_table_id} \
  --subnet-id ${web_subnet_id_1}

aws ec2 associate-route-table \
  --route-table-id ${internet_route_table_id} \
  --subnet-id ${web_subnet_id_2}
```

# EC2 security groups on the VPC
```bash
# SG for the public web servers
web_sg_id="$(aws ec2 create-security-group \
  --group-name 'SG-webserver' \
  --description 'Web servers security group' \
  --vpc-id ${my_vpc_id} \
  --query 'GroupId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${web_sg_id} \
  --tags Key=Name,Value='SG-WebServers'

# Allow ssh from my IP
aws ec2 authorize-security-group-ingress \
  --group-id ${web_sg_id} \
  --protocol 'tcp' \
  --port 22 \
  --cidr ${my_cidr}

# Allow http/s from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id ${web_sg_id} \
  --protocol 'tcp' \
  --port ${http_port} \
  --cidr ${all_cidr}

aws ec2 authorize-security-group-ingress \
  --group-id ${web_sg_id} \
  --protocol 'tcp' \
  --port ${https_port} \
  --cidr ${all_cidr}

# SG for the private app servers
app_sg_id="$(aws ec2 create-security-group \
  --group-name 'SG-appserver' \
  --description 'App servers security group' \
  --vpc-id ${my_vpc_id} \
  --query 'GroupId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${app_sg_id} \
  --tags Key=Name,Value='SG-AppServers'

# Allow ssh, http/s from the web servers
aws ec2 authorize-security-group-ingress \
  --group-id ${app_sg_id} \
  --protocol 'tcp' \
  --port 22 \
  --source-group ${web_sg_id}

aws ec2 authorize-security-group-ingress \
  --group-id ${app_sg_id} \
  --protocol 'tcp' \
  --port ${http_port} \
  --source-group ${web_sg_id}

aws ec2 authorize-security-group-ingress \
  --group-id ${app_sg_id} \
  --protocol 'tcp' \
  --port ${https_port} \
  --source-group ${web_sg_id}

# VPC SG for the RDS instances
db_sg_id="$(aws ec2 create-security-group \
  --group-name 'SG-dbserver' \
  --description 'DB servers security group' \
  --vpc-id ${my_vpc_id} \
  --query 'GroupId' | tr -d  '"')"

aws ec2 create-tags \
  --resources ${db_sg_id} \
  --tags Key=Name,Value='SG-DBServers'

# Allow MySQL/Aurora access from the app servers
aws ec2 authorize-security-group-ingress \
  --group-id ${db_sg_id} \
  --protocol 'tcp' \
  --port ${mysql_port} \
  --source-group ${app_sg_id}
```

# RDS
```bash
rds_subnet_group_name='rdssubnetgroup'

# Specify a db subnet group that defines which subnets in that VPC can be used
# for db instances.
aws rds create-db-subnet-group \
  --db-subnet-group-name ${rds_subnet_group_name} \
  --db-subnet-group-description 'Subnet group for RDS' \
  --subnet-ids ${db_subnet_id_1} ${db_subnet_id_2}
```

## RDS Aurora
Aurora stores copies of the data in a DB cluster in a logical volume across multiple AZs in a single region, regardless of whether the instances in the DB cluster span multiple AZs.
```bash
# (Optional) create a custom:
#   db cluster parameter group (container for engine config)

aurora_cluster_id='my-aurora-cluster'
aurora_primary_inst_id='my-aurora-instance'
aurora_read_replica_inst_id='my-aurora-read-replica'
aurora='aurora'

# Create Aurora cluster
aws rds create-db-cluster \
  --db-cluster-identifier ${aurora_cluster_id} \
  --engine ${aurora} \
  --master-username ${rds_master_user} \
  --master-user-password ${rds_master_password} \
  --db-subnet-group-name ${rds_subnet_group_name} \
  --vpc-security-group-ids ${db_sg_id} \
  --database-name 'mydb'

# (Optional) create a custom:
#   db parameter group (container for engine config)
#   option group (additional engine-specific features)

# --publicly-accessible | --no-publicly-accessible

# Create Aurora primary intance - read/write
aws rds create-db-instance \
  --db-cluster-identifier ${aurora_cluster_id} \
  --db-instance-identifier ${aurora_primary_inst_id} \
  --db-instance-class ${rds_aurora_inst_type} \
  --engine ${aurora} \
  --db-name 'mydb' \
  --availability-zone ${az_1} \
  --no-publicly-accessible

# Create Aurora read replica(s). Can be promoted to primary in failover.
aws rds create-db-instance \
  --db-instance-identifier ${aurora_read_replica_inst_id} \
  --db-cluster-identifier ${aurora_cluster_id} \
  --engine ${aurora} \
  --db-instance-class ${rds_aurora_inst_type} \
  --availability-zone ${az_2} \
  --no-publicly-accessible \
```

## RDS MySQL/MariaDB/PostgreSQL/Oracle/SQL Server
Multi-AZ deployment:
- Primary
- Standby for failover
- Read replica(s)

```bash
# mysql | mariadb | oracle-se1 | oracle-se2 | oracle-se | oracle-ee |
# sqlserver-ee | sqlserver-se | sqlserver-ex | sqlserver-web | postgres | aurora

# --publicly-accessible | --no-publicly-accessible

# --multi-az | --no-multi-az
# Note: You can't set the AZ parameter if MultiAZ is true

# (Optional) create a custom:
#   db parameter group (container for engine config)
#   option group (additional engine-specific features)

rds_primary_inst_id='my-rds-instance'
rds_read_replica_inst_id='my-read-replica'
mysql='mysql'

# Create primary instance (and standby replica if MultiAZ)
aws rds create-db-instance \
  --db-instance-identifier ${rds_primary_inst_id} \
  --db-instance-class ${rds_inst_type} \
  --allocated-storage ${rds_alloc_storage} \
  --engine ${mysql} \
  --vpc-security-group-ids ${db_sg_id} \
  --db-name 'mydb' \
  --db-subnet-group-name ${rds_subnet_group_name} \
  --multi-az \
  #--availability-zone ${az_1} \
  --no-publicly-accessible \
  --master-username ${rds_master_user} \
  --master-user-password ${rds_master_password}

# Create read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier ${rds_read_replica_inst_id} \
  --source-db-instance-identifier ${rds_primary_inst_id} \
  --availability-zone ${az_2}
```

# DynamoDB
Data is automatically replicaded across AZs. The AWS CLI can interact with DynamoDB running on your computer.  
To enable this, add the --endpoint-url parameter to each command:  
--endpoint-url http://localhost:8000

When you provision a table you specify:
1. Primary key
- Partition (hash) key.
- Partition (hash) and Sort (range) key. Allows for secondary indexes.
2. Provisioned capacity in read/write units

Secondary indexes  
- Global: Has partition and sort key that can be different from the table's.  
Can be created/deleted at any time.  
The table can have multiple.  
Has its own capacity units.  
- Local: Has same partition key, different sort key.  
Can be created only when you create the table.  
Can't be removed or modified.  
The table can only have one.  
Item updates consume write capacity units from the table.  

## Run it locally
```bash
java -Djava.library.path=./DynamoDBLocal_lib -jar DynamoDBLocal.jar \
  -sharedDb -inMemory

# Create a table
aws dynamodb create-table \
    --endpoint-url 'http://localhost:8000' \
    --table-name 'Music' \
    --key-schema 'AttributeName=Artist,KeyType=HASH' \
                 'AttributeName=SongTitle,KeyType=RANGE' \
    --provisioned-throughput 'ReadCapacityUnits=1,WriteCapacityUnits=1' \
    --attribute-definitions \
        'AttributeName=Artist,AttributeType=S' \
        'AttributeName=SongTitle,AttributeType=S'

# Add items
aws dynamodb put-item \
  --endpoint-url 'http://localhost:8000' \
  --table-name 'Music'  \
  --item \
      '{"Artist": {"S": "No One You Know"}, "SongTitle": {"S": "Call Me Today"}, "AlbumTitle": {"S": "Somewhat Famous"}}' \
  --return-consumed-capacity 'TOTAL'

aws dynamodb put-item \
  --endpoint-url 'http://localhost:8000' \
  --table-name 'Music'  \
  --item '{ "Artist": {"S": "Acme Band"}, "SongTitle": {"S": "Happy Day"}, "AlbumTitle": {"S": "Songs About Life"}}' \
  --return-consumed-capacity 'TOTAL'

aws dynamodb put-item \
  --endpoint-url 'http://localhost:8000' \
  --table-name 'Music'  \
  --item '{ "Artist": {"S": "Acme Band"}, "SongTitle": {"S": "Woof"}, "AlbumTitle": {"S": "Songs About Dogs"}}' \
  --return-consumed-capacity 'TOTAL'

aws dynamodb query \
  --endpoint-url 'http://localhost:8000' \
  --table-name 'Music' \
  --key-conditions 'file://key-conditions.json'

aws dynamodb query \
  --endpoint-url 'http://localhost:8000' \
  --table-name 'Music' \
  --key-conditions 'file://partitionkey-conditions.json'
```

```bash
################################################################################
# TODO: SQS, SNS, SWF
################################################################################


################################################################################
# TODO: Create IAM policies for S3 buckets, DynamoDB
#
# Examples:
#
# Policy for reading a config file
#
# {
#     "Version": "2012-10-17",
#     "Statement": [
#         {
#             "Sid": "Stmt1465609541000",
#             "Effect": "Allow",
#             "Action": [
#                 "s3:GetObject"
#             ],
#             "Resource": [
#                 "arn:aws:s3:::myconfigbucket/test.env.js"
#             ]
#         }
#     ]
# }
#
# Policy for getting the application from a deployment bucket
#
# {
#     "Version": "2012-10-17",
#     "Statement": [
#         {
#             "Sid": "Stmt1465794395000",
#             "Effect": "Allow",
#             "Action": [
#                 "s3:ListBucket"
#             ],
#             "Resource": [
#                 "arn:aws:s3:::mydistbucket"
#             ]
#         },
#         {
#             "Sid": "Stmt1465794443000",
#             "Effect": "Allow",
#             "Action": [
#                 "s3:GetObject"
#             ],
#             "Resource": [
#                 "arn:aws:s3:::mydistbucket/*"
#             ]
#         }
#     ]
# }

################################################################################
# TODO: Roles for instances that need access to S3, DynamoDB using policies above
################################################################################
```

# Key pairs
```bash
my_key_pair='myawskey'

aws ec2 create-key-pair \
  --key-name ${my_key_pair} \
  --query 'KeyMaterial' | tr -d  '"' > "${my_key_pair}.pem"
```

# Finding images
```bash
# amazon | aws-marketplace | microsoft
# hvm | paravirtual
# x86_64 | i386
# ebs | instance-store
# available | pending | failed
# json | table | text

aws ec2 describe-images --filters \
  Name=owner-alias,Values=amazon \
  Name=virtualization-type,Values=hvm \
  Name=architecture,Values=x86_64 \
  Name=root-device-type,Values=ebs \
  Name=state,Values=available \
  Name=description,Values='Amazon Linux AMI 2016*' \
  --output table \
  > aws-ebs-images.txt

aws ec2 describe-images --filters \
  Name=owner-alias,Values=amazon \
  Name=virtualization-type,Values=hvm \
  Name=architecture,Values=x86_64 \
  Name=root-device-type,Values=instance-store \
  Name=state,Values=available \
  Name=description,Values='Amazon Linux AMI 2016*' \
  --output table \
  > aws-is-images.txt

aws ec2 describe-reserved-instances-offerings \
  --product-description 'Linux/UNIX (Amazon VPC)' \
  --instance-type ${web_inst_type} \
  --availability-zone ${az_1}
```

```bash
################################################################################
# TODO: App server instances
################################################################################

# Create instances. Include User Data to deploy app from S3. If there's an error
# copying from S3 specify the --region.

# Load balancer

################################################################################
# TODO: Web server instances
################################################################################

```

# Create instances. 
Include User Data to deploy app from S3. If there's an error copying from S3 specify the --region.
```bash
# --associate-public-ip-address | --no-associate-public-ip-address
aws ec2 run-instances \
  --image-id ${AMZN_LINUX_AMI_ID} \
  --count 1 \
  --instance-type ${web_inst_type} \
  --key-name ${my_key_pair} \
  --security-group-ids ${web_sg_id} \
  --subnet-id ${web_subnet_id_1} \
  --associate-public-ip-address

# Load balancer

################################################################################
# TODO: Autoscaling
################################################################################
# Create Launch Configuration
# Create CloudWatch alarms for the auto scaling groups
# Create Auto Scaling Groups and tag them

################################################################################
# TODO: Bastion hosts
################################################################################

```

# Network Address Translation (NAT)
## NAT instance 
Good for dev/test. Create an instance in each AZ for high availability. You'll need to manage failover/autoscaling groups in each AZ.
```bash
nat_instance_id="$(aws ec2 run-instances \
  --image-id ${AMZN_LINUX_NAT_AMI_ID} \
  --count 1 \
  --instance-type ${web_inst_type} \
  --key-name ${my_key_pair} \
  --security-group-ids ${web_sg_id} \
  --subnet-id ${web_subnet_id_1} \
  --associate-public-ip-address \
  --query 'Instances[0].InstanceId' | tr -d  '"')"

aws ec2 modify-instance-attribute \
  --instance-id ${nat_instance_id}
  --no-source-dest-check

main_route_table_id="$(aws ec2 describe-route-tables \
  --filters 'Name=association.main,Values=true' \
            "Name=vpc-id,Values=${my_vpc_id}" \
  --query 'RouteTables[0].RouteTableId' | tr -d  '"')"

aws ec2 create-route \
  --route-table-id ${main_route_table_id} \
  --destination-cidr-block ${all_cidr} \
  --instance-id ${nat_instance_id}
```

## NAT Gateway
Good for staging/production. Place a NAT Gateway in each AZ for a zone-independent architecture. A gateway in an AZ has redundancy built-in.
```bash
ip_alloc_id="$(aws ec2 allocate-address \
  --domain 'vpc' \
  --query 'AllocationId' | tr -d  '"')"

nat_gtw_id="$(aws ec2 create-nat-gateway \
  --subnet-id ${web_subnet_id_1} \
  --allocation-id ${ip_alloc_id} \
  --query 'NatGateway.NatGatewayId'| tr -d  '"')"

aws ec2 create-route \
  --route-table-id ${main_route_table_id} \
  --destination-cidr-block ${all_cidr} \
  --nat-gateway-id ${nat_gtw_id}
```

# CloudFormation
json templates have parameters, outputs with "Fn::GetAtt". Rollback by default.
