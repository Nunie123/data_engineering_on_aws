# Up and Running: Setting Up Your Data Engineering Infrastructure on AWS
The completely free E-Book for setting up and running a Data Engineering stack on AWS and Snowflake.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the [Contributions section](https://github.com/Nunie123/data_engineering_on_aws#user-content-contributions).

## Table of Contents
[Introduction](https://github.com/Nunie123/data_engineering_on_aws) <br>
**Chapter 1: Setting Up Your Accounts** (You are here) <br>
[Chapter 2: Data Warehousing with Snowflake](https://github.com/Nunie123/data_engineering_on_aws/blob/main/02_data_warehousing.md) <br>
Chapter 3: Infrastructure as Code with Terraform and Terragrunt <br>
Chapter 4: Streaming Data with Snowpipe and AWS Kinesis <br>
Chapter 5: Orchestrating Pipelines with Airflow and AWS MWAA <br>
Chapter 6: Transforming Data with dbt <br>
Chapter 7: Processing Data with AWS Batch <br>
Chapter 8: Event-Driven Pipelines with AWS Lambda and SNS <br>
Chapter 9: Deployment Pipelines with GitHub Actions <br>
Chapter 10: Alerting with AWS CloudWatch and Airflow

---

# Chapter 1: Setting Up Your Accounts

This chapter will walk you through setting up your accounts and your development environment for completing the tasks described in the rest of this book. In particular, we will be doing the following:
1. Signing up for an AWS Account
2. Signing up for a Snowflake Account
3. Signing up for a GitHub Account
4. Setting up your development environment
   1. Installing Git
   2. Installing Python 3
   3. Installing Terraform
   4. Installing Terragrunt
   5. Installing dbt
   6. Installing AWS CLI

Note that we will be using features we have to pay for in AWS, Snowflake, and GitHub. All of these offer free options with limits, but they do not offer a way to automatically turn off the service when the free limits are exceeded. You should be able to follow along with this book with only a small bill from these services (less than $20). But be aware that if you leave certain services running for extended periods your bill could be much higher. 

## Signing Up for an AWS Account

By signing up for a new account we will be eligible for AWS Free Tier, which allows us to use many AWS services for free. If you do not create a new account, all of the services you use will need to be paid for out of pocket. 

We will need to provide AWS with an email address that is not already associated with an AWS account, our phone number, our address, and our credit card information.

Follow the prompts here to sign up: https://aws.amazon.com/console/

We aren't going to be using AWS until Chapter 3. Be sure to save your AWS login credentials for later.

## Signing Up for a Snowflake Account

We are going to be using Snowflake as our data warehouse. To sign up for the free trial all we need is to provide our email address. Follow the prompts here: https://www.snowflake.com/.

After signing up we'll be brought to Snowflake's web console showing a few example databases pre-populated with data. We can leave this alone for now, but will be returning to this console in Chapter 2. Be sure to bookmark this URL, as you will need it to navigate back to your warehouse.

## Signing Up for a GitHub Account

If you don't already have a GitHub Account, you can create one here: https://github.com/. In Chapter 9 we'll discuss how to use GitHub Actions to perform automated deployments to AWS.

We'll need three repositories in GitHub for this code, which I'm calling:
1. data_engineering_on_aws_code for our main code base
2. data_engineering_on_aws_tf for our Terraform Modules code
3. data_engineering_on_aws_tg for our Terragrunt code

Once we're logged in to GitHub we can click the dropdown with our profile image in the upper right and select "Your repositories". From there we can select the "New" button, which will bring us to a form for creating your repository. Provide your repository's name and then select "Create repository". Don't worry about the other options on this page.

We will be brought to a page with a link to our repository that looks similar to this: `https://github.com/Nunie123/data_engineering_on_aws_code.git`. Copy that link.

Our repository is now created on GitHub, but we need to clone it to our local machine. Open the terminal and navigate to a folder where we want this repository to be saved. Now execute `git clone` followed by the link we copied in the previous step. So for me, the command is:
``` Bash
git clone https://github.com/Nunie123/data_engineering_on_aws_code.git
```
If you get a `command not found` error after executing this command, then follow the instructions from the **Installing Git** section, below.

You should receive a warning about cloning an empty repository, but don't worry about it. You now have a folder on your machine with the same name as your GitHub repository. This folder is a local git repository which is linked to your remote git repository on GitHub. In later chapters we'll be talk more about how to use git, including how to integrate it into our automated deployment pipelines.

Repeat the above steps for the remaining two repositories. We should now have three git repositories on our computer linked to three remote repositories on GitHub.


## Setting Up Your Development Environment

To build our data engineering infrastructure we will need to execute some commands from the command line. The code provided in this book was tested using a terminal running `zsh` on macOS. We need access to the terminal so that we can run command line utilities like the AWS CLI, Terragrunt, and dbt. I will cover how to install these utilities if you are on macOS. If you are a linux machine you'll have to do your own research on how to install these tools. However, the commands provided as sample code in later chapters should be the same, regardless of whether you are using a command line on macOS vs. Linux. 

This book does not cover the Command Prompt nor PowerShell on Windows.

#### Installing Git

Let's start by installing Git, a command line utility for version control that will also allow us to interact with GitHub. First, let's see if you already have Git installed by executing:
``` Bash
git --version
```
If you don't have git installed you will be prompted to install git and the Xcode Command Line Tools. Follow the prompts to complete the installation. Xcode can take awhile to install, so go get a snack. More information on installing git is [here](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

#### Installing Python 3

Next let's install Python 3. Python comes in two major versions: Python 2 and Python 3. macOS comes with Python 2 by default, despite the majority of Python development now being done in Python 3. We can see if you have Python 3 installed by executing:
``` Bash
python3 --version
```
If you get a `command not found` error, you'll need to install Python 3. If you have [Homebrew](https://brew.sh/) installed on you machine, then installing Python 3 is as easy as:
``` Bash
brew install python
```
You can also download Python 3 from their [website](https://www.python.org/downloads/macos/).

#### Installing Terraform

Terraform is a command line utility that is part of our Infrastructure as Code solution. You can install it with Homebrew using:
``` Bash
brew update
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```
You can also install Terraform manually, as described [here](https://learn.hashicorp.com/tutorials/terraform/install-cli).

#### Installing Terragrunt

While we could Terraform by itself to define our AWS infrastructure, Terragrunt is nice because it cuts down on boilerplate code and makes our infrastructure code more [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).

Your best bet for installing Terragrunt is with Homebrew:
``` Bash
brew install terragrunt
```

We'll start working with Terragrunt and Terraform in Chapter 3.

#### Installing dbt
dbt is a tool for transforming data in our data warehouse. You can install it with Homebrew:
``` Bash
brew tap dbt-labs/dbt
brew install dbt
```

You can also install it with `pip`, the package manager for Python:
``` Bash
pip3 install dbt
```

We'll start using dbt in Chapter 6.

#### Installing AWS CLI

We want the AWS CLI (version 2) so that we can interact with various AWS services directly from the command line. To install it, you can run: 
``` Bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```
You can also install it through AWS' GUI installer, as described [here](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

---

Next Chapter: [Chapter 2: Data Warehousing with Snowflake](https://github.com/Nunie123/data_engineering_on_aws/blob/main/02_data_warehousing.md)