## Overview


EC2 instances left running overnight cost money. We're going to figure out how to turn them on and off automatically
every day; and our stretch goal is a manual on-switch. Cost management: Begin!

Follow this 
[documentation link](https://aws.amazon.com/premiumsupport/knowledge-center/start-stop-lambda-cloudwatch/)
for the instructions. Here on this page it is a "personal narrative". So first I established a cloud account 
and then I logged in as an **admin** User. 


## Procedural notes

> Suggestion: Read through these notes, then go through the procedural at the above link.


A Lambda function is serverless code that runs on AWS in response to a trigger of some sort. 
We will build **Start** and **Stop** Lambda functions that will be triggered by a daily CloudWatch
alarm.

> ***IMPORTANT WARNING:*** If you already have an EC2 **work instance** running that includes work you value: 
> *Do Not Wire These Lambda Functions Into That Instance!!!*  Something could go wrong. Instead set up
> a test EC2 instance and only when you are satisfied everything is working properly should you re-point
> the Lambda functions at your work instance. And to be super-safe, back up your work instance to an
> Amazon Machine Image (AMI). Then if something goes awry you will be able to reconstitute your work instance.

## Starting an EC2


Here are notes on all seven console-wizard start steps for Launching an EC2.


* Region: Before starting the Launch wizard: Select a nearby Region (upper right), in my case US-West-2 Oregon
* Image (OS): Ubuntu Server 64-bit x86; noting this is a choice of an AMI
    * The Quick Start is one of four tabs, the others being My AMIs, AWS Marketplace and Community AMIs
    * Question for Joel: Talk about these other tabs, how they might jump start a project
* Instance Type: **c4.large** as a cheap test VM
* Configure Instance... here we use strictly Default values; and I add notes on "What is this option good for?"
    * Number: 1
    * Spot: No (but this leads to another cost management avenue)
    * Network: Which VPC to use
        * For Joel: At what point do you hit this and think 'Definitely need a custom VPC for this instance?
    * Subnet: Default
        * For Joel: What drives choice of sub-net? Do these map to AZs? What about *private* subnets? Can we do those here?
    * Auto-assign public IP: Default
        * For Joel: What do these options do?
    * Hostname type: Default
        * For Joel: What options do we have here? Does the Resource Name mean 'human-friendly URL' possible?
    * DNS Hostname: Default
        * For Joel: Again what are the options here?
    * Placement group: Un-checked
        * For Joel: What is the function of a Placement group? 
        * **Cluster**, **Partition** and **Spread** seem to correspond to cluster computing and two types of failsafe...
        * Could use further explanation. Also how do Placement Groups interact with VPCs? 
    * Capacity Reservation: Open
        * For Joel: What does this do?
    * Domain join directory: No directory
        * For Joel: When do I need one? AT built braininfo.org; do these persist as zombies? 
    * IAM Role: None
        * For Joel: Many are listed in the drop-down. Zero cost zombies?
        * Cleaning up old Roles: Can we look at one and ask "What resources have this Role?"
    * CPU Options: Un-checked
        * For Joel: When checking this are we overriding the instance specs from above?
    * Shutdown behavior: Stop
        * For Joel: Terminate option is for disposable use cases... elaborate?
    * Stop-Hibernate behavior (unchecked): What is this?
    * Enable Termination Protection (unchecked): Slows down careless terminations?
    * Monitoring (unchecked): What does CloudWatch cost? Need a heuristic and a value
    * EBS-optimized: Greyed out. Depends on instance type? What does this optimization do?
    * Tenancy: Shared. Cost of dedicated? Value of dedicated?
    * Elastic Inference (unchecked): Function? Cost? Value?
    * File system (un-used): What does this do? 







## Prior version of these notes here down

* I created the necessary policy in the JSON editor (under IAM), named it `stop-instances`, tagged it so I know what it does
* The IAM *Role* for pending Lambdas is `stop-instances`
    * Notice the above *Policy* is in the *Role*. The *Role* is unassociated at this point
        * A *Policy* is a text file that spells out AWS logic for what is permitted
        * A *Role* is an entity that carries a *Policy*. In our case: The *Role* is an automated alarm entity
* **Stop-Lambda** works...
    * In the code source window: click on the left sidebar `lambda_function.py` to activate the editor for this file.


> **IMPORTANT:** Make sure to adjust both region and instance values to match your implementation


> **IMPORTANT:** The instance IDs *should* be imported from environment variables to make this code more secure.


* Fix: Do the import env vars for fully detailed walk-through that included the code
    * Timeout is under the Configuration tab (not the default Code tab).
    * Save the code and be sure to **Deploy** it

* Creating the CloudWatch alarm trigger
    * Name, description, choose `Schedule` and `Cron expression`
    * My cron format is `0  5  ?  *  3-7  *` which translates as "5 AM Zulu every Tuesday through Saturday" (10AM PDT)
        * Days of the week are 1-7 SUN-SAT; but it uses less-than logic
        * The ? wildcard means use days of week, not days of month (3rd field)
    * After the alarm set: Return to the Lambda function and did a refresh: Now the CloudWatch alarm appears as a trigger
        * I could have but did not do this from within the Lambda Function page

Here is the code. To do: Change the information to reside in environment variables.

```
import boto3
region = 'us-west-2'
instances = ['i-aaabbbcccdddeeeff']
ec2 = boto3.client('ec2', region_name=region)

def lambda_handler(event, context):
    print('start stopping instances')
    ec2.stop_instances(InstanceIds=instances)
    print('done stopping instances')
```
