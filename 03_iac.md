# Up and Running: Setting Up Your Data Engineering Infrastructure on AWS
The completely free E-Book for setting up and running a Data Engineering stack on AWS and Snowflake.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the [Contributions section](https://github.com/Nunie123/data_engineering_on_aws#user-content-contributions).

## Table of Contents
[Introduction](https://github.com/Nunie123/data_engineering_on_aws) <br>
[Chapter 1: Setting Up Your Accounts](https://github.com/Nunie123/data_engineering_on_aws/blob/main/01_accounts.md) <br>
[Chapter 2: Data Warehousing with Snowflake](https://github.com/Nunie123/data_engineering_on_aws/blob/main/02_data_warehousing.md) <br>
**Chapter 3: Infrastructure as Code with Terraform and Terragrunt** (You are here) <br>
[Chapter 4: Streaming Data with Snowpipe and AWS Kinesis](https://github.com/Nunie123/data_engineering_on_aws/blob/main/04_streaming.md) <br>
Chapter 5: Orchestrating Pipelines with Airflow and AWS MWAA <br>
Chapter 6: Transforming Data with dbt <br>
Chapter 7: Processing Data with AWS Batch <br>
Chapter 8: Event-Driven Pipelines with AWS Lambda and SNS <br>
Chapter 9: Deployment Pipelines with GitHub Actions <br>
Chapter 10: Alerting with AWS CloudWatch and Airflow

---

# Chapter 3: Infrastructure as Code with Terraform and Terragrunt

As Data Engineers we have to set up and maintain a lot of infrastructure. By setting up an Infrastructure as Code (IaC) system we get several benefits:
* Infrastructure configurations are put under version control.
* Infrastructure can undergo the same code review process as all our other code.
* Setting up new infrastructure is simpler.
* We can easily roll back to a previous configuration.
* Disaster recovery is simplified.

There are lots of IaC tools out there, but we are going to be using Terraform and Terragrunt. These tools are both popular and cloud provider agnostic (i.e. we could also use them to set up infrastructure on GCP, Azure, etc.).

## An Overview of Terraform and Terragrunt

Terraform allows us to create what are essentially configuration files (with a `.tf`) extension that define our infrastructure on AWS. We can then have Terraform apply these configurations to our AWS account, resulting in our infrastructure being built.

Because we are also using Terragrunt, we will create Terraform files that will act as templates, so we can have a single Terraform template for building all of our S3 buckets, for example. We will then have a seperate file for each bucket instance, that Terragrunt will apply to the template and then instantiate in AWS.

## Setting Up Terraform

In Chapter 1 we created a repo called `data_engineering_on_aws_tf`. This is the repo that will contain our Terraform templates.

In the next chapter we'll be setting up a data pipeline to stream data from our fictional company's application database. So for now we can use Terraform and Terragrunt to provision that RDS database.

Inside our Terraform repo let's make a folder called `rds`. And inside the `rds` folder let's create three files: `provider.tf`, `variables.tf`, and `rds.tf`.

So our folder structure looks like this:
```
data_engineering_on_aws_tf
    |_ rds
        |_ provider.tf
        |_ rds.tf
        |_ variables.tf
```

Inside the provider file we will indicate we want to use AWS in region us-west-2. We do that by adding this code to the file:
``` javascript
provider "aws" {
  region = "us-west-2"
}
```

The code that Terraform uses to identify what infrastructure to build is inside the `rds.tf` file. Because these are essentially configuration files you should always look at the documentation to understand what settings to provide. The documentation for RDS is here: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance.

``` javascript
resource "aws_db_instance" "rds_db" {
  allocated_storage                 = var.allocated_storage
  max_allocated_storage             = var.max_allocated_storage
  engine                            = var.engine
  engine_version                    = var.engine_version
  instance_class                    = var.instance_class
  name                              = var.name
  username                          = "master"
  password                          = random_password.db_master_pass.result
  parameter_group_name              = aws_db_parameter_group.defaault.id
  skip_final_snapshot               = var.skip_final_snapshot
  enabled_cloudwatch_logs_exports   = var.enabled_cloudwatch_logs_exports
  db_subnet_group_name              = aws_db_subnet_group.subnet_group.id
  vpc_security_group_ids            = var.security_group_ids
  identifier                        = var.name

  tags = {
    Environment = var.environment
  }
}


resource "random_password" "db_master_pass" {
  length            = 40
  special           = true
  min_special       = 5
  override_special  = "!#$%^&*()-_=+[]{}<>:?"
  keepers           = {
    pass_version  = 1
  }
}


resource "aws_secretsmanager_secret" "db-pass" {
  name = "db-master-password-${var.name}-${var.environment}"
}


resource "aws_secretsmanager_secret_version" "db-pass-val" {
  secret_id     = aws_secretsmanager_secret.db-pass.id
  secret_string = random_password.db_master_pass.result
}


resource "aws_db_parameter_group" "default" {
  name   = "rds-${var.name}-pg"
  family = "mysql8.0"
}


resource "aws_db_subnet_group" "subnet_group" {
  name       = "${var.name}_db_subnet_group"
  subnet_ids = var.subnet_ids

  tags = {
    Environment = var.environment
  }
}

```

You'll notice that Terraform code will create six resources, five of which will be deployed to AWS:
1. An AWS RDS instance.
2. A local random password.
3. An AWS Secrets Manager secret to hold the password that was generated.
4. A secret version that adds the password that was generated to the secret in AWS.
5. An RDS parameter group for further configuring the MySQL server running on RDS.
6. An RDS subnet group which identifies the VPC the RDS instance will be a part of.

Finally, let's populate the `variables.tf` file with the code that identifies what parameters our `rds.tf` file needs:
``` javascript
variable "environment" {}
variable "allocated_storage" {}
variable "max_allocated_storage" {}
variable "engine" {}
variable "engine_version" {}
variable "instance_class" {}
variable "name" {}
variable "skip_final_snapshot" {
    type = bool
}
variable "enabled_cloudwatch_logs_exports" {
    type = list
}
variable "subnet_ids" {
    type = list
}
variable "security_group_ids" {
    type = list
}
```

This RDS instance requires that we provide IDs for the subnets and security groups that will be associated with that instance. So let's also write some Terraform code to set up our VPC and related networking configurations. Just like the "rds" folder, we'll create a new folder called "vpc" at the top level of our repo. We'll create three files: `provider.tf`, `vpc.tf`, and `variables.tf`. Now our repo should look like this:
```
data_engineering_on_aws_tf
    |_ rds
        |_ provider.tf
        |_ rds.tf
        |_ variables.tf
    |_ vpc
        |_ provider.tf
        |_ vpc.tf
        |_ variables.tf
```
Documentation for configuring a VPC through Terraform is here: . Let's fill out our three files as shown:

`provider.tf`
``` javascript
provider "aws" {
  region = "us-west-2"
}
```

`vpc.tf`
``` javascript
resource "aws_vpc" "vpc" {
  cidr_block            = var.vpc_cidr_block
  enable_dns_hostnames  = true

  tags = {
    Environment = var.environment
    Name        = var.vpc_name
  }
}

resource "aws_nat_gateway" "nat" {
  allocation_id       = aws_eip.eip.allocation_id
  subnet_id           = aws_subnet.public_subnet_1.id
  connectivity_type   = var.connectivity_type
  tags = {
    Environment = var.environment
    Name        = "NAT Gateway for ${var.vpc_name}"
 }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.vpc.id

  tags = {
    Environment = var.environment
    Name        = "IGW for ${var.vpc_name}"
  }
}

resource "aws_subnet" "public_subnet_1" {
  vpc_id     = aws_vpc.vpc.id
  cidr_block = var.public_subnet_1_cidr_block
  map_public_ip_on_launch = true
  availability_zone = var.subnet_availability_zone_1

  tags = {
    Environment = var.environment
    Name        = "public_subnet_1"
  }
}

resource "aws_subnet" "public_subnet_2" {
  vpc_id     = aws_vpc.vpc.id
  cidr_block = var.public_subnet_2_cidr_block
  map_public_ip_on_launch = true
  availability_zone = var.subnet_availability_zone_2

  tags = {
    Environment = var.environment
    Name        = "public_subnet_2"
  }
}

resource "aws_subnet" "private_subnet_1" {
  vpc_id     = aws_vpc.vpc.id
  cidr_block = var.private_subnet_1_cidr_block
  map_public_ip_on_launch = false
  availability_zone = var.subnet_availability_zone_1

  tags = {
    Environment = var.environment
    Name        = "private_subnet_1"
  }
}

resource "aws_subnet" "private_subnet_2" {
  vpc_id     = aws_vpc.vpc.id
  cidr_block = var.private_subnet_2_cidr_block
  map_public_ip_on_launch = false
  availability_zone = var.subnet_availability_zone_2

  tags = {
    Environment = var.environment
    Name        = "private_subnet_2"
  }
}

resource "aws_route_table" "route_table_public" {
  vpc_id = aws_vpc.vpc.id

  route {
      cidr_block = "0.0.0.0/0"
      gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Environment = var.environment
    Name        = "Route table for public subnet"
  }
}

resource "aws_route_table" "route_table_private" {
  vpc_id = aws_vpc.vpc.id

  route {
      cidr_block = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.nat.id
  }

  tags = {
    Environment = var.environment
    Name        = "Route table for private subnet"
  }
}

resource "aws_route_table_association" "route_association_public_1" {
  subnet_id      = aws_subnet.public_subnet_1.id
  route_table_id = aws_route_table.route_table_public.id
}

resource "aws_route_table_association" "route_association_private_1" {
  subnet_id      = aws_subnet.private_subnet_1.id
  route_table_id = aws_route_table.route_table_private.id
}

resource "aws_route_table_association" "route_association_public_2" {
  subnet_id      = aws_subnet.public_subnet_2.id
  route_table_id = aws_route_table.route_table_public.id
}

resource "aws_route_table_association" "route_association_private_2" {
  subnet_id      = aws_subnet.private_subnet_2.id
  route_table_id = aws_route_table.route_table_private.id
}

resource "aws_network_acl" "acl_public" {
  vpc_id = aws_vpc.vpc.id
  subnet_ids = [aws_subnet.public_subnet_1.id, aws_subnet.public_subnet_2.id]

  egress {
      # Allow all
      protocol   = "all"
      rule_no    = 100
      action     = "allow"
      cidr_block = "0.0.0.0/0"
      from_port  = 0
      to_port    = 0
    }

  ingress {
    # Allow all
    protocol   = "all"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = {
    Environment = var.environment
    Name        = "ACL for public subnets"
  }
}


resource "aws_network_acl" "acl_private" {
  vpc_id = aws_vpc.vpc.id
  subnet_ids = [aws_subnet.private_subnet_1.id, aws_subnet.private_subnet_2.id]

  egress {
    # Allow all
    protocol   = "all"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  ingress {
    # Allow all
    protocol   = "all"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
  tags = {
    Environment = var.environment
    Name        = "ACL for private subnets"
  }
}


resource "aws_eip" "eip" {
  vpc      = true
  tags = {
    Environment = var.environment
    Name        = "EIP for NAT Gateway"
  }
}
```

`variables.tf`
``` javascript
variable "vpc_cidr_block" {}
variable "environment" {}
variable "public_subnet_1_cidr_block" {}
variable "public_subnet_2_cidr_block" {}
variable "private_subnet_1_cidr_block" {}
variable "private_subnet_2_cidr_block" {}
variable "subnet_availability_zone_1" {}
variable "subnet_availability_zone_2" {}
variable "connectivity_type" {
  default = "public"
}
variable "vpc_name" {}
```

There is a lot of network configuration code in there. As networking is not the focus of this book, I'm not going to discuss what each piece of infrastructure we're setting up does. Suffice to say that when we deploy our VPC it will allow us to create infrastructure within it that is exposed to the internet, as well as infrastructure that is not exposed.

Finally we need to push these new files to our repo:
``` Bash
git status
git add --all
git commit -m "Adding TF code for RDS and VPC"
git push
```

## Saving Terraform State in S3

For Terraform to work properly it needs to maintain a state file. This file keeps track of all the infrastructure Terraform is responsible for. Terraform needs to know this so that if we tell Terraform to apply a change, and the resource already exists, it won't try to make a duplicate instance of the resource. Also, by tracking this state on it's own Terraform can keep track of the infra being managed by Terraform, and ignore the infrastructure that is being managed outside of Terraform.

We are going to save our state file in S3, so that it can be easily shared if multiple people are working in Terraform. But in order to do that we'll need to create an S3 bucket. Let's go ahead and create that bucket through the AWS S3 console:

1. Log into the AWS console, which can be done here: https://aws.amazon.com/console/
2. Go to the S3 page in the AWS console and select "Create Bucket"
3. Provide a name for your bucket (I'm using "data-engineering-on-aws-terraform") and leave all the other settings alone. Click "Create Bucket" at the bottom.

We've created our first piece of AWS infrastructure: a bucket we will use to hold our Terraform state file. Remember this bucket name, as we'll need it in the next section when we configure Terragrunt to find and update this state file. Because we created this bucket through the console, rather than through Terraform, we can't manage this bucket through Terraform, unless we later import it into Terraform.

## Setting Up Terragrunt

We've set up our Terraform repo, which will act as our templates. And we've set up our S3 bucket to store our state file. Now we need to actually build our infrastructure, and that's going to be done with Terragrunt.

Let's navigate to our Terragrunt repo and add two folders, one called `vpc` and the other called `rds`. Inside `vpc` we'll make a folder called `main_vpc`. Inside `rds` we'll create a folder called `company_application_db`. Inside each of the inner folders we need to add two files, called `terragrunt.hcl` and `main.tf`. Your Terragrunt repo should now have this structure:
```
data_engineering_on_aws_tg
    |_vpc
        |_main.tf
        |_terragrunt.hcl
    |_rds
        |_main.tf
        |_terragrunt.hcl
```

We'll deploy our VPC (Virtual Private Cloud) first, so we can put our RDS instance inside of it. Populate the files in the `vpc` folder like so:

`vpc/main.tf`
``` javascript
terraform {
  backend "s3" {}
}
```

`vpc/terragrunt.hcl`
``` javascript
remote_state {
    backend = "s3"
    config = {
        bucket = "data-engineering-on-aws-terraform"

        key = "${path_relative_to_include()}/vpc/terraform.tfstate"
        region         = "us-west-2"
        encrypt        = true
    }
}
  terraform {
    source = "git::git@github.com:Nunie123/data_engineering_on_aws_tf.git//vpc?ref=master"

    extra_arguments "conditional_vars" {
      commands = [
        "apply",
        "plan",
        "import",
        "push",
        "refresh",
        "destroy"
      ]

    }
}

inputs = {
    environment = "dev"
    vpc_cidr_block = "192.168.0.0/16"
    public_subnet_1_cidr_block = "192.168.1.0/24"
    public_subnet_2_cidr_block = "192.168.2.0/24"
    private_subnet_1_cidr_block = "192.168.3.0/24"
    private_subnet_2_cidr_block = "192.168.4.0/24"
    subnet_availability_zone_1 = "us-west-2a"
    subnet_availability_zone_2 = "us-west-2b"
    connectivity_type = "public"
    vpc_name = "vpc_data_engineering_on_aws"
}
```

This Terragrunt code is providing our configuration variables to our VPC Terraform code.

If you're running this on your own machine you need to make two changes:
1. Update the line `source = "git::git@github.com:Nunie123/data_engineering_on_aws_tf.git//vpc?ref=master"` to reference your own git repo.
2. Update the bucket name at `bucket = "data-engineering-on-aws-terraform"` to reference your bucket.

All the other configuration settings should work for your environment, but let briefly describe what each do:
* `environment` is just being used to tag our infrastructure, and has no functional meaning (as far AWS is concerned).
* The `..._cidr_block` variables all refer to the internal IP Addresses are going to use. There's a lot of detail on how networking works that is outside the scope of this book. Check out [Amazon's documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html) if you want to learn a bit more about how it all works.
* The `...availability_zone...` variables refer to which sections within "us-west-2" we want our VPC to run in.
* `connectivity_type` indicates whether you want to allow unsolicited inbound connections from the internet.
* `vpc_name` is used to tag the name of the VPC.

Now that we've added our Terragrunt code to our repo let's deploy it. To deploy to AWS you'll need to authernticate using the AWS Access Key ID and AWS Secret Access Key that you saved in Chapter 1. In the terminal add your keys to the environment by executing:
``` Bash
export AWS_ACCESS_KEY_ID=<your key>
export AWS_SECRET_ACCESS_KEY=<you secret key>
export AWS_DEFAULT_REGION=us-west-2
```

Now let's navigate to the `vpc` folder in our Terragrunt repo in the command line and execute:
``` Bash
terragrunt apply
```
If this is your first time connecting to GitHub over SSH you may receive a message like:
``` text
The authenticity of host 'github.com (192.30.255.113)' can't be established.
ED25519 key fingerprint is SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```
Type `yes` and hit return.

If you get a `git@ssh.github.com: Permission denied (publickey)` error at this point, you probably just need to get SSH access set up for GitHub. The directions for setting that up are [here](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/checking-for-existing-ssh-keys).

A Terraform Plan will now be printed out showing all the resources that will be created. You should get this prompt at the bottom:
``` Bash
Plan: 16 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```
Type `yes` and hit return.

It'll take a couple minutes for everything to be built, so feel free to take a break. You should hopefully see `Apply complete! Resources: 16 added, 0 changed, 0 destroyed.` printed in your terminal.

Now let's do the same process for RDS. But before we can complete our Terragrunt code we need to get the Subnet and Security Group IDs from AWS. 

1. We can view our VPCs [here](https://us-west-2.console.aws.amazon.com/vpc/home?region=us-west-2#vpcs:). Note the VPC ID for later.
2. View your Subnets [here](https://us-west-2.console.aws.amazon.com/vpc/home?region=us-west-2#subnets:sort=VpcId). Note the Subnet ID for `private_subnet_1` and `private_subnet_2`.
3. View your Security Groups [here](https://us-west-2.console.aws.amazon.com/vpc/home?region=us-west-2#securityGroups:). A default Security Group was set up when you created your new VPC. Note the ID.

Now we have what we need to fill out our files for Terragrunt:

`rds/main.tf`
``` javascript 
terraform {
  backend "s3" {}
}
```

`rds/terragrunt.hcl`
``` javascript
remote_state {
    backend = "s3"
    config = {
        bucket = "data-engineering-on-aws-terraform"

        key = "${path_relative_to_include()}/rds/terraform.tfstate"
        region         = "us-west-2"
        encrypt        = true
    }
}
  terraform {
    source = "git::git@github.com:Nunie123/data_engineering_on_aws_tf.git//rds?ref=master"

    extra_arguments "conditional_vars" {
      commands = [
        "apply",
        "plan",
        "import",
        "push",
        "refresh",
        "destroy"
      ]

    }
}

inputs = {
	environment = "dev"
    allocated_storage = 20
    max_allocated_storage = 100
    engine = "mysql"
    engine_version = "8.0"
    instance_class = "db.m6g.large"
    name = "applicationdb"
    skip_final_snapshot = true
    enabled_cloudwatch_logs_exports = ["audit", "error", "general", "slowquery"]
    subnet_ids = ["subnet-052ba433585f0126c", "subnet-00eae6a4fd096b4df"]
    security_group_ids = ["sg-0ce1572ed6042edc2"]
}

```

Navigate to your `rds` folder in your Terragreunt repo and repeat the deploy process you did for your VPC:
``` Bash
terragrunt apply
```

## Wrapping Up

We now have a VPC and RDS instance that we've deployed using our Infrastructure as Code system. More importantly, we now have the code in place to easily deploy and take down VPC and RDS infrastructure. In future chapters we will add more Terraform code to support new types of infrastructure.

We are going to be using the infrastructure we built in this chapter in the naxt chapter. So if you plan on moving directly to the next chapters then you can leave the VPC and RDS instance running. But if it is going to be awhile before you do more work on this book, you may want to take this infrastructure down. It's generally a good practice to take down infrastructure you are not using, and it will reduce your AWS bill.

Taking down the infrastructure is easy. On the command line from within the Terragrunt repo navigate to the folder where your `terragrunt.hcl` file is located and execute `terragrunt destroy`. It will prompt you to confirm, so type `yes` and hit return. Do that for each `terragrunt.hcl` file and all of your infrastructure will be taken down.

When you need to bring this infrastructure back up, simply use `terragrunt apply`, as described above.

Now that our Terragrunt code is ready, let's add it to our repo:
``` Bash
git status
git add --all
git commit -m "Adding TG code for RDS and VPC"
git push
```

---
Next Chapter: [Chapter 4: Streaming Data with Snowpipe and AWS Kinesis](https://github.com/Nunie123/data_engineering_on_aws/blob/main/04_streaming.md) <br>