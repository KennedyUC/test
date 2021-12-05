![](https://avatars.githubusercontent.com/u/13208838?s=200&v=4)

# **AWS SQS with S3 Integration** 

This documentation is about the steps to setup AWS SQS with S3 integration. The process flow is such that when a file drops in the S3 bucket, it triggers a notification to AWS SQS queue.

The infrastructures are provisioned using terraform.

To achieve this, the following procedures must be completed:

1. Configuring AWS EKS Cluster
1. Configuring AWS S3 bucket
1. Configuring AWS SQS queue
1. Integrating the configured modules
1. Testing the workflow

The major resources are configured in a modularized manner. This makes it possible for the resources to be easily troubleshooted and reused.

The modules are configured in different repositories and finally integrated in a repository.  
  
When everything is set and the final terraform module is called, with the parameters passed, the configured AWS infrastructures will be created in the specified AWS user account (based on the inputed AWS access and secret keys).  
  
You just have to login to the AWS user console to work with the configured infrastructure.  

The modules are configured in four major terraform files in each of the different repositories.  
  
- **Main terraform file**  
  This contains the main set of configurations for the module. The resources in the module are configured in this file. The resources are configured by declaring the resource type and the resource name in the block header. The configuration arguments are placed within the body of the block and enclosed by { }. A combination of the resource type and name serve as a unique identifier to a given resource within the module.  

-  **Providers terraform file**:  
This is where the provider of the required infrastructures is defined.

- **Output terraform file**:  
This is where the output parameters are defined.

- **Variable terraform file**:  
The variables in the module are defined in this file.  

When executed, terraform runs through these files in an alphabetical order and the infrastructures will be created.


# **Integrating the AWS S3 Module and SQS Module**
The configured AWS EKS Cluster, S3-Bucket and SQS-queue are integrated in a repository by calling them as modules using their respective repo addresses.
  
| Name      | Version |
|-----------|---------|
| Terraform | 0.14.0+ |
| AWS       | 3.0+    |

| Provider | Version |
|------|---------|
| AWS  | 3.0+    |
 
  
| Module      | Source                                         |
|-------------|------------------------------------------------|
| EKS Cluster | https://github.com/MavenCode/terraform-aws-eks |
| AWS S3      | https://github.com/MavenCode/terraform-aws-s3  |
| AWS SQS     | http://github.com/MavenCode/terraform-aws-sqs  |  


| Input       | Description                                                                   |
|-------------|-------------------------------------------------------------------------------|
| access_key  | The unique key used to get programmatic access to the AWS resources               |
| secret_key  | The unique key used to get programmatic access to the AWS resources               |
| region      | The AWS region to send the request, based on the users specification  |
| environment | Devlopment, production or staging environment                                  |
| vpc_id      | The virtual private cloud where the EKS cluster will be deployed |  
  
| Output                 | Description                   |
|------------------------|-------------------------------|
| s3_bucket notification | Bucket for notification queue |

## **Main Terraform File**
### **Bootstrapping the AWS EKS Cluster Module**  
The configured terraform-aws-eks module is called by referencing the repository address ([aws-eks module](http://github.com/MavenCode/terraform-aws-eks "aws-eks module")).  

        module "bootstrap-eks-cluster" {
            source     = "github.com/MavenCode/terraform-aws-eks"
            access_key = var.access_key
            secret_key = var.secret_key

            eks_cluster_name = var.eks_cluster_name
            region           = var.region
            env              = var.env
            vpc_id           = var.vpc_id

            ip_ranges     = var.ip_ranges
            subnet_ids    = var.subnet_ids
            instance_type = var.instance_type
            desired_size  = var.desired_size
            max_size      = var.max_size

            min_size = var.min_size
        }
- The *access and secret keys* are set  
These are generated from the AWS user console. These keys are used to get programmatic access to the AWS resources.
    
- The *region* is defined  
This specifies the AWS region to send the request, based on the users specification. The default region is set as us-east-2 in the root module.

- The *environment* is defined   
This can be developement or production environment

- The *VPC_id* is set:  
This defines the virtual private cloud where the EKS cluster will be deployed  

- The *IP_ranges* is set  
This defines the IP ranges the AWS use.

- The *subnet_ids* is set  
These provides a set of ids for a vpc_id

---

### **Bootstrapping the S3 module**  
The configured terraform-aws-s3 module is called by referencing the repository address ([aws-s3 module](http://github.com/MavenCode/terraform-aws-s3 "aws-s3 module")).


        module "s3_bucket" {
            source = "github.com/MavenCode/terraform-aws-s3"
            bucket_name = "tf-s3-bucket"
            access_key = ""
            secret_key = ""
            acl = "public-read" 
        }
- The bucket_name is defined

- The access and secret keys are set to access the AWS S3 resource in the AWS user account.  
  
- The network *access control list* (ACL) is set.  
  It provides a layer of security, and acts as a firewall for controlling traffic in and out of one or more subnets.  

---

### **Bootstrapping the SQS module**
The configured terraform-aws-sqs module is called by referencing the repository address ([aws-sqs module](http://github.com/MavenCode/terraform-aws-sqs "aws-sqs module")).  

        module "sqs" {
            source = "github.com/MavenCode/terraform-aws-sqs"
            name   = "s3-event-notification-queue"

            delay_seconds             = var.delay_seconds
            max_message_size          = var.max_message_size
            message_retention_seconds = var.message_retention_seconds
            receive_wait_time_seconds = var.receive_wait_time_seconds 

            policy = <<POLICY
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": "*",
                    "Action": "sqs:SendMessage",
                    "Resource": "arn:aws:sqs:*:*:s3-event-notification-queue",
                    "Condition": {
                    "ArnEquals": { "aws:SourceArn": "${aws_s3_bucket.bucket.arn}" }
                    }
                }
            ]
        }
        POLICY
        }  

- The *delay_seconds* is set  
This indicates the time, in seconds, it takes for the delivery of all notification messages in the queue will be delayed. An integer from 0 to 900  
  
- The *max_message_size* is set  
This represents the maximum bytes of a notification message before AWS SQS rejects it.  
  
- The *message_retention_seconds* is set  
This represents the number of seconds AWS SQS retains a notification message.  

- The *receive_wait_time_seconds* is set  
This represents the time for which a ReceiveMessage call will wait for a message to arrive (long polling) before returning.   It is defined by an integer from 0 to 20 (seconds)  
  
- The *policy* is defined within the block of code.
---

### **S3 notification for message events configuration**

        resource "aws_s3_bucket_notification" "bucket_notification" {
            bucket = aws_s3_bucket.bucket.id

            queue {
                queue_arn = module.sqs.queue.arn
                events    = ["s3:ObjectCreated:*"]
            }
        }  
        
- The *aws_s3_bucket_notification* is defined  
- The queue is configured by defining the queue_arn and the trigger events.

---
## **Output Terraform File**
The desired output is configured in this file.

        output "notification" {
            value = aws_s3_bucket_notification.bucket_notification.queue[0]
        }
- The output value is set to s3_bucket notification

---

## **Provider Terraform File**
The provider of the infrastructure is configured in this file.

        terraform {
            required_providers {
                aws = {
                    source = "hashicorp/aws"
                    version = "3.66.0"
                }
            }
        }


        provider "aws" {
            region = var.region
            access_key = var.access_key
            secret_key = var.secret_key
        }
- The provider is defined as "aws"  
- The access and secret keys are defined as variables.  
  

## **Testing the Workflow**  
Github action is used to run the integrated terraform module directly from the repository.

#still in progress, awaiting the github action to be created in the repository  
#which houses the final integrated module.
