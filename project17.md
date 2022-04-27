# Project 17 Automate Infrastructure With IAC using Terraform Part 2 (Storage and Database) # 


## Create Elastic File System (EFS) ##

In order to create an EFS you need to create a KMS key.

AWS Key Management Service (KMS) makes it easy to create and manage cryptographic keys and control their use across a wide range of 
AWS services and applications.

Create efs.tf file and use this code 
~~~
# create key from key management system
resource "aws_kms_key" "PRJ17-kms" {
  description = "KMS key"
  policy      = <<EOF
  {
  "Version": "2012-10-17",
  "Id": "kms-key-policy",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::${var.account_no}:user/terraform" },
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
EOF
}

# create key alias
resource "aws_kms_alias" "alias" {
  name          = "alias/kms"
  target_key_id = aws_kms_key.PRJ17-kms.key_id
}
~~~

Next, create EFS and it's mount targets- add the following code to *efs.tf*

~~~
# set first mount target for the EFS 
resource "aws_efs_mount_target" "subnet-1" {
  file_system_id  = aws_efs_file_system.PRJ17-efs.id
  subnet_id       = aws_subnet.private-webs[0].id
  security_groups = [aws_security_group.allow_webservers-sg.id]
}

# set second mount target for the EFS 
resource "aws_efs_mount_target" "subnet-2" {
  file_system_id  = aws_efs_file_system.PRJ17-efs.id
  subnet_id       = aws_subnet.private-webs[1].id
  security_groups = [aws_security_group.allow_webservers-sg.id]
}

# create access point for wordpress
resource "aws_efs_access_point" "wordpress" {
  file_system_id = aws_efs_file_system.PRJ17-efs.id

  posix_user {
    gid = 0
    uid = 0
  }

  root_directory {
    path = "/wordpress"

    creation_info {
      owner_gid   = 0
      owner_uid   = 0
      permissions = 0755
    }

  }

}

# create access point for tooling
resource "aws_efs_access_point" "tooling" {
  file_system_id = aws_efs_file_system.PRJ17-efs.id
  posix_user {
    gid = 0
    uid = 0
  }

  root_directory {

    path = "/tooling"

    creation_info {
      owner_gid   = 0
      owner_uid   = 0
      permissions = 0755
    }

  }
}
~~~

The above adds *access points* for tooling and Wordpress to the EFS


### Create MySQL RDS ###

Create the rds.tf file, put the code below in the file

~~~
# This section will create the subnet group for the RDS  instance using the private subnet
resource "aws_db_subnet_group" "PRJ17-rds" {
  name       = "prj17-rds"
  subnet_ids = [aws_subnet.private-data[0].id, aws_subnet.private-data[1].id]

 tags = merge(
    var.tags,
    {
      Name = "PRJ17-rds"
    },
  )
}

# create the RDS instance with the subnets group
resource "aws_db_instance" "PRJ17-rds" {
  allocated_storage      = 20
  storage_type           = "gp2"
  engine                 = "mysql"
  engine_version         = "5.7"
  instance_class         = "db.t2.micro"
  db_name                = "bayodb"
  username               = var.master-username
  password               = var.master-password
  parameter_group_name   = "default.mysql5.7"
  db_subnet_group_name   = aws_db_subnet_group.PRJ17-rds.name
  skip_final_snapshot    = true
  vpc_security_group_ids = [aws_security_group.allow_webservers-sg.id]
  multi_az               = "true"
}
~~~

At this point, the *variables-tf* file should look like this:

~~~
variable "region" {
        default = "us-east-1"
    }
    
variable "vpc_cidr" {
        default = "172.16.0.0/16"
    }

variable "enable_dns_support" {
        default = "true"
    }

variable "enable_dns_hostnames" {
        default ="true" 
    }

variable "enable_classiclink" {
        default = "false"
    }

variable "enable_classiclink_dns_support" {
        default = "false"
    }
    
variable "account_no" {
  type        = number
  description = "the account number"
}

variable "master-username" {
  type        = string
  description = "RDS admin username"
}

variable "master-password" {
  type        = string
  description = "RDS master password"
}

variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}


variable "ami" {
  type        = string
  description = "AMI ID for the launch template"
}

variable "keypair" {
  type        = string
  description = "key pair for the instances"
}

variable "preferred_number_of_public_subnets" {
    default = 2
}

variable "preferred_number_of_private-webs_subnets" {
    default = 2
}


variable "preferred_number_of_private-data_subnets" {
    default = 2
}

~~~

Use
~~~
terraform validate
~~~
To check the validity of the codes so far

![](terraform-validate.jpg)

The *terraform.tfvars* file should look like this:

~~~
region = "us-east-1"

vpc_cidr = "172.16.0.0/16" 

enable_dns_support = "true" 

enable_dns_hostnames = "true"  

enable_classiclink = "false" 

enable_classiclink_dns_support = "false" 

preferred_number_of_public_subnets = 2

preferred_number_of_private-webs_subnets = 2

preferred_number_of_private-data_subnets = 2

environment = "production"

ami = "ami-0b0af3577fe5e3532"

keypair = "evops"

# Ensure to change this to your acccount number
account_no = "760200903907"

master-username = "bayo"

master-password = "bayodevops"

tags = {
  Enviroment      = "production" 
  Owner-Email     = "badedeji@gmail.com"
  Managed-By      = "Terraform"
  Billing-Account = "760200903907"
}
~~~

**Additional tasks**
1. **IP Address**: This is a means of uniquely identifying a computer host on a network. There are two versions currently in use, IPV4 and IPV6. IPV4 is a 32bit address written in dot notation like this 172.17.16.1, depending on the cidr used, a part of the address refers to the network portion while the other part describes the host part. IPV6 on the other hand is 128bit long, meaning there are 2 raise to the power of 128 (trillion trillion trillion possible addresses) It is written in hex format as against IPV4 which is written in decimal format. 00ab::45EA:1234::0A::0 
1. **Subnets**: A subnet is a logical delineation of a larger network to seperate network host for security and other network management reasons. So a big network can besub-divided into smaller ones. 10.0.0.0/24 (256 addresses) can be divided into 10.0.0.0/25 and 10.0.0.128/23 (126 addresses each)
1. **CIDR Notation**: Was invented to further extend the availability of IPV4 when it was noticed that it will be exhausted soon. It helped to cut IPV$ wastages when a network/ organisation is assigned too large IPV4 space then required under old Classful allocation method. A network that needs 20 IPs shouldn't be given a class C block (256 address) with CIDR, a /27 (30 possible addresses) can be allocated
1. **IP Routing**: IP routing is the process of moving IP packets across networks. 
1. **Internet**: The internet is the interconnection of computers worldwide. It was developed from the concept of LANs (Local Area Network) which was primarily to share resources rather than duplicate resources like printers etc. The development of TCP/IP protocol made the emmergence of the internet as we know it today easier. 
1. **Gateways**: Gatways are  devices that allow traffic to pass through from one network to the other. Routers are gateways.
1. **NAT**: Net Address Translation was designed to slow down the exhaustion of IPV4 addresses, but allowing multiple private IPV4 addresses access the internet using a single public address. All the traffic from the private IPs are represented by the public IPV4
