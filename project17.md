# Project 17 Automate Infrastructure With IAC using Terraform Part 2 # 

## Private Subnets ##
Create two groups of Private subnets
1. Webservers Private Subnet (one per AZ)
1. atalayer Private Subnet (one per AZ)


Make sure you use variables or *length()* function to determine the number of AZs
Use variables and *cidrsubnet()* function to allocate vpc_cidr for subnets
Keep variables and resources in separate files for better code structure and readability
Tags all the resources you have created so far. Explore how to use format() and count functions to automatically tag subnets with its respective number.

### A little bit more about Tagging ###
Tagging is a straightforward, but a very powerful concept that helps you manage your resources much more efficiently:

Resources are much better organized in ‘virtual’ groups
- They can be easily filtered and searched from console or programmatically
- Billing team can easily generate reports and determine how much each part of infrastructure costs how much (by department, by type, by environment, etc.)
- You can easily determine resources that are not being used and take actions accordingly
- If there are different teams in the organisation using the same account, tagging can help differentiate who owns which resources.

Now you can tag all you resources using the format below
~~~
tags = merge(
    var.tags,
    {
      Name = "Name of the resource"
    },
  )
~~~

NOTE: Update the variables.tf to declare the variable tags used in the format above;
~~~
variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
~~~
The nice thing about this is – anytime we need to make a change to the tags, we simply do that in one single place (terraform.tfvars).

### Internet Gateways & format() function ###
Create an Internet Gateway in a separate Terraform file internet_gateway.tf
~~~
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.PRJ16-vpc.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-%s!", aws_vpc.main.id,"IG")
    } 
  )
}
~~~
notice how we have used format() function to dynamically generate a unique name for this resource? The first part of the %s takes the interpolated value of aws_vpc.main.id while the second %s appends a literal string IG and finally an exclamation mark is added in the end.

If any of the resources being created is either using the count function, or creating multiple resources using a loop, then a key-value pair that needs to be unique must be handled differently.

For example, each of our subnets should have a unique name in the tag section. Without the format() function, we would not be able to see uniqueness. With the format function, each private subnet’s tag will look like this.
~~~
Name = PrvateSubnet-0
Name = PrvateSubnet-1
Name = PrvateSubnet-2
Lets try and see that in action.
~~~

~~~
  tags = merge(
    var.tags,
    {
      Name = format("PrivateSubnet-%s", count.index)
    } 
  )
~~~

### NAT Gateways ###
Create 1 NAT Gateways and 1 Elastic IP (EIP) addresses

Now use similar approach to create the NAT Gateways in a new file called *natgateway.tf*.

Note: We need to create an Elastic IP for the NAT Gateway, and you can see the use of depends_on to indicate that the Internet Gateway resource must be available before this should be created. Although Terraform does a good job to manage dependencies, but in some cases, it is good to be explicit

~~~
resource "aws_eip" "prj17-nat-eip" {
  vpc        = true
  depends_on = [aws_internet_gateway.ig]

  tags = merge(
    var.tags,
    {
      Name = format("%s-EIP", var.name)
    },
  )
}

resource "aws_nat_gateway" "prj17-nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.ig]

  tags = merge(
    var.tags,
    {
      Name = format("%s-Nat", var.name)
    },
  )
}
~~~

Run to validate the codes so far
~~~
sudo terraform validate
~~~
![](validate.jpg)


### AWS ROUTES ###
Create a file called route_tables.tf and use it to create routes for both public and private subnets, create the below resources. Ensure they are properly tagged.

- aws_route_table
- aws_route
- aws_route_table_association

~~~
# create webserver private route table
resource "aws_route_table" "webs-private-rtb" {
  vpc_id = aws_vpc.PRJ16-vpc.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Private-Route-Table", var.name)
    },
  )
}

# associate webs private subnets to the private route table
resource "aws_route_table_association" "webs-private-subnets-assoc" {
  count          = length(aws_subnet.private-webs[*].id)
  subnet_id      = element(aws_subnet.private-webs[*].id, count.index)
  route_table_id = aws_route_table.webs-private-rtb.id
}


# create datalayer private route table
resource "aws_route_table" "data-private-rtb" {
  vpc_id = aws_vpc.PRJ16-vpc.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Private-Route-Table", var.name)
    },
  )
}

# associate data private subnets to the private route table
resource "aws_route_table_association" "data-private-subnets-assoc" {
  count          = length(aws_subnet.private-data[*].id)
  subnet_id      = element(aws_subnet.private-data[*].id, count.index)
  route_table_id = aws_route_table.data-private-rtb.id
}

# create route table for the public subnets
resource "aws_route_table" "public-rtb" {
  vpc_id = aws_vpc.PRJ16-vpc.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-Public-Route-Table", var.name)
    },
  )
}
# create route for the public route table and attach the internet gateway
resource "aws_route" "public-rtb-route" {
  route_table_id         = aws_route_table.public-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}

# associate all public subnets to the public route table
resource "aws_route_table_association" "public-subnets-assoc" {
  count          = length(aws_subnet.public[*].id)
  subnet_id      = element(aws_subnet.public[*].id, count.index)
  route_table_id = aws_route_table.public-rtb.id
  }
~~~
We now have:

– [x] Our main vpc
– [x] 1 Public subnets
– [x] 4 Private subnets
– [x] 1 Internet Gateway
– [x] 1 NAT Gateway
– [x] 1 EIP
– [x] 3 Route tables

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
