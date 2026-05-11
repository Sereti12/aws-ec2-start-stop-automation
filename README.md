# Automated EC2 Instance Scheduling
## Using AWS Lambda and Amazon EventBridge

<p align="center">
  <img src="Architecture.png" alt="Architecture Diagram" width="1000"/>
</p>

*Figure 1: High-level architecture — Amazon EventBridge triggers an AWS Lambda function to start and stop tagged EC2 instances on a defined schedule.* 

This project demonstrates hands-on experience in automating the scheduled start and stop of Amazon EC2 instances. The instances are automatically started each morning before business hours begin and stopped at the end of the working day. The primary objective is cost optimization: EC2 instances that run continuously outside of business hours incur unnecessary charges. By implementing automated scheduling, organizations can significantly reduce compute costs without requiring manual intervention from cloud engineers.
The solution achieves the following operational goals:
*	Eliminate idle EC2 compute costs outside of business hours.
*	Remove the operational overhead of manually starting and stopping instances.
* Enforce consistent uptime schedules across all tagged instances.
* Maintain full auditability through CloudWatch logging.

The automation is implemented using a serverless, event-driven architecture leveraging Amazon EventBridge for scheduling and AWS Lambda for execution. EC2 instances are identified using resource tags, making the solution scalable and easily extensible to additional instances without modifying the underlying code.
