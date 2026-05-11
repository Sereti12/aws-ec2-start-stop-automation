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
#### 1.2  Creating the Subnet
A public subnet was created within the VPC using the following settings:
* Subnet name: Public-Subnet-1
* Availability Zone: Europe (Stockholm) — eu-north-1a
*	VPC IPv4 CIDR block: 10.0.0.0/16
*	Subnet IPv4 CIDR block: 10.0.1.0/24
The /24 subnet provides 256 IP addresses (251 usable), which is appropriate for this workload.

<p align="center">
  <img src="fig2.png.png" alt="Architecture Diagram" width="1000"/>
</p>

*Figure 2: Subnet configuration showing the subnet name, Availability Zone, VPC CIDR block, and subnet CIDR block.*

#### 1.3  Enabling Auto-Assign Public IP
Auto-assign Public IP was enabled on the subnet so that instances launched into it automatically receive a public IPv4 address, enabling outbound internet connectivity.
#### 1.4  Creating and Attaching the Internet Gateway
An Internet Gateway (UAT-IGW) was created and attached to UAT-VPC. The Internet Gateway allows resources within the public subnet to communicate with the public internet.
#### 1.5  Adding the Internet Route
A route was added to the VPC route table with the following configuration:
* Destination: 0.0.0.0/0 (all internet traffic)
* Target: UAT-IGW (the Internet Gateway)
This route ensures that all outbound traffic from the subnet is directed to the Internet Gateway.

<p align="center">
  <img src="fig3.png.png" alt="Architecture Diagram" width="1000"/>
</p>

*Figure 3: Route table entry directing all internet-bound traffic (0.0.0.0/0) to the Internet Gateway.*
#### 1.6  Associating the Route Table with the Public Subnet
The route table containing the internet route was associated with Public-Subnet-1, completing the public subnet configuration. Instances in this subnet can now route traffic to and from the internet.
### Step 2: Security Group Configuration
A security group (UAT-Web-SG) was created to act as a virtual firewall for the EC2 instances. The following inbound rules were configured:

| Type  | Protocol | Port Range | Source                     |
|-------|-----------|-------------|----------------------------|
| SSH   | TCP       | 22          | My IP (restricted)         |
| HTTP  | TCP       | 80          | 0.0.0.0/0 (Anywhere)       |
| HTTPS | TCP       | 443         | 0.0.0.0/0 (Anywhere)       |

**Note:** SSH access is restricted to "My IP" rather than "Anywhere" (0.0.0.0/0). This is an important security best practice, as it limits SSH access to only the engineer's current IP address and prevents unauthorized remote access attempts from the public internet.

<p align="center">
  <img src="fig4.png.png" alt="Architecture Diagram" width="1000"/>
</p>

*Figure 4: Security group inbound rules showing SSH restricted to My IP, with HTTP and HTTPS open to all sources.*

### Step 3: Launching EC2 Instances
Two EC2 instances were launched into the public subnet using the following configuration:
* AMI: Amazon Linux 2023
* Instance type: t3.micro

<p align="center">
  <img src="fig5.png.png" alt="Architecture Diagram" width="1000"/>
</p>

*Figure 5: Instance configuration showing the selected AMI (Amazon Linux 2023), instance type (t3.micro), and architecture.*

The following network settings were applied during launch:
| Resource                | Value            |
|-------------------------|------------------|
| VPC                     | UAT-VPC          |
| Subnet                  | Public-Subnet-1  |
| Auto-assign Public IP   | Enabled          |
| Security Group          | UAT-Web-SG       |

<p align="center">
  <img src="fig6.png.png" alt="Architecture Diagram" width="1000"/>
</p>

*Figure 6: Network settings applied during EC2 instance launch, including VPC, subnet, public IP assignment, and security group.*

### Step 4: Tagging EC2 Instances
Resource tagging is the mechanism by which the Lambda function identifies which instances to manage. All instances participating in the automation were tagged with a consistent key-value pair:
* Key: environment
* Value: UAT
Using tags as the selection mechanism makes the solution scalable. Additional instances can be included in or excluded from the automation simply by adding or removing the tag, without any changes to the Lambda code.

<p align="center">
  <img src="fig7.png.png" alt="Architecture Diagram" width="1000"/>
</p>

*Figure 7: EC2 instance tags. All instances included in the automation share the same "environment=UAT" tag.*

### Step 5: Creating the IAM Policy
An IAM policy was created to define the minimum permissions required for the Lambda function to operate. Following the principle of least privilege, the policy grants only the permissions necessary for the automation workflow and no more.

<p align="center">
  <img src="fig8.png.png" alt="Architecture Diagram" width="1000"/>
</p>

*Figure 8: IAM policy shown in the policy editor, defining EC2 management and CloudWatch Logs permissions.*

**The Policy**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:Start*",
        "ec2:Stop*",
        "ec2:DescribeInstanceStatus"
      ],
      "Resource": "*"
    },
    {
      "Sid": "VisualEditor1",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:CreateLogGroup",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```
**Policy Explanation**

The policy contains two permission statements:

**Statement 1 — EC2 Instance Management (Sid: VisualEditor0):**
*	ec2:DescribeInstances — Allows the function to list and retrieve details of EC2 instances, which is required to resolve tag filters to instance IDs.
* ec2:Start* — Permits the function to start EC2 instances.
* ec2:Stop* — Permits the function to stop EC2 instances.
* ec2:DescribeInstanceStatus — Allows the function to query the current state of instances for validation and logging purposes.

**Statement 2 — CloudWatch Logs (Sid: VisualEditor1):**
* logs:CreateLogGroup — Allows Lambda to create a new log group if one does not already exist.
* logs:CreateLogStream — Allows Lambda to create log streams within the log group.
* logs:PutLogEvents — Allows Lambda to write log entries, enabling execution activity to be monitored and audited.

**Note:** The Resource field is set to "*" (all resources) for simplicity in this project. In a production environment, this should be scoped down to specific resource ARNs to further restrict access in accordance with the principle of least privilege.
