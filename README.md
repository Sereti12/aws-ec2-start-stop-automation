# Automated EC2 Instance Scheduling
## Using AWS Lambda and Amazon EventBridge

<p align="center">
  <img src="Architecture.png" alt="Architecture Diagram" width="1000"/>
</p>

*Figure 1: High-level architecture — Amazon EventBridge triggers an AWS Lambda function to start and stop tagged EC2 instances on a defined schedule.* 

## 1. Project Overview

This project demonstrates hands-on experience in automating the scheduled start and stop of Amazon EC2 instances. The instances are automatically started each morning before business hours begin and stopped at the end of the working day. The primary objective is cost optimization: EC2 instances that run continuously outside of business hours incur unnecessary charges. By implementing automated scheduling, organizations can significantly reduce compute costs without requiring manual intervention from cloud engineers.
The solution achieves the following operational goals:
*	Eliminate idle EC2 compute costs outside of business hours.
*	Remove the operational overhead of manually starting and stopping instances.
* Enforce consistent uptime schedules across all tagged instances.
* Maintain full auditability through CloudWatch logging.

The automation is implemented using a serverless, event-driven architecture leveraging Amazon EventBridge for scheduling and AWS Lambda for execution. EC2 instances are identified using resource tags, making the solution scalable and easily extensible to additional instances without modifying the underlying code.

**Note:** From a security standpoint, a private subnet with a NAT Gateway would be the ideal network configuration for production workloads. However, due to the additional cost of NAT Gateways — one of the more expensive AWS networking components — a public subnet was used for this project. In a production environment, the NAT Gateway approach is strongly recommended to avoid exposing instance network interfaces directly to the internet.

## 2. Architecture

This solution is built on a lightweight, serverless architecture consisting of the following AWS components:

### Amazon VPC
Provides a logically isolated, customizable virtual network within AWS. All project resources are deployed within this VPC to ensure network-level isolation and control.
### Public Subnet
A subnet within the VPC configured to route traffic to the internet via an Internet Gateway. The EC2 instances are placed in this subnet for this project.
### Security Group
Acts as a stateful virtual firewall for the EC2 instances, controlling inbound and outbound traffic at the instance level based on defined rules.
### Amazon EventBridge
A serverless event bus that enables the creation of fine-grained cron-based schedules. In this solution, EventBridge serves as the trigger mechanism, invoking the Lambda function at configured start and stop times.
### AWS Lambda
A serverless compute service that executes the Python-based automation code in response to EventBridge triggers. Lambda requires no server provisioning or management and charges only for actual execution time.
### IAM Role and IAM Policy
The IAM role grants the Lambda function a temporary, least-privilege identity to interact with AWS services. The attached IAM policy explicitly defines the permitted actions: describing, starting, and stopping EC2 instances, as well as publishing logs to Amazon CloudWatch.

**The workflow operates as follows:**
1.	Amazon EventBridge evaluates the configured cron schedule.
2.	At the designated time, EventBridge invokes the Lambda function and passes a JSON payload specifying the action ("start" or "stop").
3.	The Lambda function authenticates using its attached IAM role and retrieves EC2 instances matching the configured tag filter.
4.	The function calls the appropriate EC2 API action on the identified instances.
5.	Execution logs, including matched instances and action results, are published to Amazon CloudWatch Logs for monitoring and auditability.

## 3. Prerequisites
Before implementing this solution, ensure the following requirements are met:
* An active AWS account with access to the AWS Management Console.
* Basic to intermediate familiarity with core AWS concepts, including VPC, EC2, IAM, Lambda, and EventBridge.
* A pre-configured VPC with at least one subnet.
* A minimum of two EC2 instances to participate in the automation.
* Familiarity with JSON for writing IAM policies and EventBridge event payloads.
* Basic understanding of Python (for reviewing and customizing the Lambda function).

## 4. Implementation
### Step 1: Networking — VPC and Subnet Configuration
#### 1.1  Creating the VPC
An Amazon VPC was created with the following configuration:
*	Name tag: UAT-VPC
*	IPv4 CIDR block: 10.0.0.0/16
The /16 CIDR block provides up to 65,536 IP addresses, offering sufficient address space for future growth.
### 1.2  Creating the Subnet
A public subnet was created within the VPC using the following settings:
* Subnet name: Public-Subnet-1
* Availability Zone: Europe (Stockholm) — eu-north-1a
*	VPC IPv4 CIDR block: 10.0.0.0/16
*	Subnet IPv4 CIDR block: 10.0.1.0/24
The /24 subnet provides 256 IP addresses (251 usable), which is appropriate for this workload.



