# Up and Running: Setting Up Your Data Engineering Infrastructure on AWS
The completely free E-Book for setting up and running a Data Engineering stack on AWS and Snowflake.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the [Contributions section](https://github.com/Nunie123/data_engineering_on_aws#user-content-contributions).

## Table of Contents
**Introduction** (You are here) <br>
[Chapter 1: Setting Up Your Accounts](https://github.com/Nunie123/data_engineering_on_aws/blob/main/01_accounts.md) <br>
[Chapter 2: Data Warehousing with Snowflake](https://github.com/Nunie123/data_engineering_on_aws/blob/main/02_data_warehousing.md) <br>
Chapter 3: Infrastructure as Code with Terraform and Terragrunt <br>
Chapter 4: Streaming Data with Snowpipe and AWS Kinesis <br>
Chapter 5: Orchestrating Pipelines with Airflow and AWS MWAA <br>
Chapter 6: Transforming Data with DBT <br>
Chapter 7: Processing Data with AWS Batch <br>
Chapter 8: Event-Driven Pipelines with AWS Lambda and SNS <br>
Chapter 9: Deployment Pipelines with GitHub Actions <br>
Chapter 10: Alerting with AWS CloudWatch and Airflow

---

# Introduction
This is a book designed to teach you how to set up and maintain a production-ready data engineering stack using [Amazon Web Services (AWS)](https://aws.amazon.com/) and [Snowflake](https://www.snowflake.com/).

Other technologies we'll be using in this book include:
* [Terraform](https://www.terraform.io/)
* [Terragrunt](https://terragrunt.gruntwork.io/)
* [Apache Airflow](https://airflow.apache.org/)
* [DBT](https://www.getdbt.com/)
* [GitHub Actions](https://github.com/features/actions)
* [Python](https://www.python.org/)
* [SQL](https://en.wikipedia.org/wiki/SQL)
* [Git](https://git-scm.com/)
* A variety of AWS services, including:
  * Batch
  * Lambda
  * Cloudwatch
  * MWAA
  * VPC
  * RDS
  * Kinesis
  * Secrets Manager
  * ECR

Each chapter will build on the previous chapter, setting up additional systems that rely on the systems set up in the previous chapter. By the end of the book you will have a complete Data Engineering infrastructure.

This book is opinionated. I've chosen a stack that has worked well for me and that I believe will work well for many data engineering teams. There is no *best* stack, but hopefully this book will introduce you to tools you can use in your work. If you think there's a better way than what I've laid out here, I'd love to hear about it. Please refer to the **Contributions** section, below.

## Who This Book Is For
This book is for people with coding familiarity that are interested in setting up professional data pipelines and data warehouses using AWS and Snowflake. I expect the readers to include:
* Data Engineers looking to learn more about AWS or Snowflake.
* Junior Data Engineers looking to learn best practices for building and working with data engineering infrastructure.
* Software Engineers, DevOps Engineers, Data Scientists, Data Analysts, or anyone else that is tasked with performing Data Engineering functions to help them with their other work.

This book assumes your familiarity with SQL and Python. If you do not have experience with these languages you'll be able to muddle through by copying the provided code, but you'll probably get more benefit taking the time to learn Python and SQL than any benefit you will get by reading this book.

This book covers a lot of ground. Many of the subjects we'll cover in just part of a chapter will have entire books written about them. While this book is comprehensive in the sense that it provides all the information you need to get a data engineering stack up and running, there is still plenty of information a Data Engineer needs to know that I've omitted from this book.

What is covered in this book is not **THE** data engineering stack, it is **A** data engineering stack. Even within AWS there are lots of ways to accomplish similar tasks. I mention this here to make sure you understand that this book is not the complete guide to being a data engineer. Rather, it is an introduction to the types of problems a data engineer solves, and a sampling common tools in AWS used to solve those problems.

Finally, there are a vast array of Data Engineering tools that are in use. I cover many popular tools for Data Engineering, but many more have been left out of this book due to brevity and my lack of experience with them. If you feel I left off something important, please read the **Contributions** section below.

## How to Read This Book
This book is divided into chapters discussing major Data Engineering concepts and tools. Each chapter builds on the knowledge and infrastructure built in the previous chapters.

If you're looking to use this book as a guide to set up your Data Engineering infrastructure from scratch, I recommend you read this book front-to-back. 

Likely, many people will find their way to this book trying to solve a specific problem (e.g. how to set up a managed Airflow instance on AWS). For these people I've tried to make each chapter as self-contained as possible. However, in some cases that's impractical. For example, when we discuss setting up alerting for Airflow DAGs we will necessarily have to have an Airflow Environment running. In those instances I will reference the previous chapter that explained how to set up the required infrastructure.

The best way to learn is by doing, which is why each chapter provides code samples. I encourage you to build this infrastructure with me, as you read through the chapters. In addition to this repo, I have three other git repositories set up that contain the code for this book:
* **data_engineering_on_aws_code** is the main code repository. It contains all the Python and SQL code used to build our pipelines and warehouse.
* **data_engineering_on_aws_tf** contains our Terraform modules, which will help us provision our infrastructure on AWS.
* **data_engineering_on_aws_tg** contains our Terragrunt code. We'll use Terragrunt and Terraform together as our Infrastructure as Code solution.

## Contributions

You may have noticed: this book is hosted on GitHub. This results in three great things:
1. The book is hosted online and freely available.
2. You can make pull requests.
3. You can create issues.

If you think the book is wrong, missing information, or otherwise needs to be edited, there are two options:
1. **Make a pull request** (the preferred option). If you think something needs to be changed, fork this repo, make the change yourself, then send me a pull request. I'll review it, discuss it with you, if needed, then add it in. Easy peasy. If you're not very familiar with GitHub, instructions for doing this are [here](https://gist.github.com/Chaser324/ce0505fbed06b947d962). If your PR is merged it will be considered a donation of your work to this project. You agree to grant a [Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) license for your work. You will be added the the **Contributors** section on this page once your PR is merged.
2. **Make an issue**. Go to the [Issues tab](https://github.com/Nunie123/data_engineering_on_aws/issues) for this repo on GitHub, click to create a new issue, then tell me what you think is wrong, preferably including references to specific files and line numbers.

I look forward to working with you all.

## Contributors
**Ed Nunes**. Ed lives in Los Angeles and works as a Data Engineer for [Sidecar Health](https://sidecarhealth.com/). Feel free to reach out to him on [LinkedIn](https://www.linkedin.com/in/ed-nunes-b0409b14/).


## License
You are free to use this book under the [Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)](https://creativecommons.org/licenses/by-nc-nd/4.0/) license.

---


Next Chapter: [Chapter 1: Setting Up Your Accounts](https://github.com/Nunie123/data_engineering_on_aws/blob/main/01_accounts.md)
