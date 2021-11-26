# Introduction


These are notes for managing cost on AWS by turning EC2 Virtual Machines on and off automatically. We proceed in three 
stages. First in this [**Introduction**](#introduction) we give a very condensed overview. 
Second in the [**Synopsis**](#synopsis) section we give a 
more extended narrative that introduces some necessary terminology / jargon. 
Thirdly we have a [**Thorough Walkthrough**](#thorough-walkthrough)
that is intended as step-by-step notes on what is going on. 


## Condensed overview


We build both a start and a stop function into the AWS cloud. These are *functions* and not *Virtual Machines*. 
They exist as code that runs without any reference to a host server; hence *serverless*. AWS calls such functions
Lambda functions. So one will do the starting and the other will do the stopping. They must both have permission
within the AWS cloud to perform this task; so they are granted *Roles* that have attached *Policies* to enable 
just that. 


In order for this task to be meaningful you must have one or more virtual machines that should regularly be 
stopped and started in an automated fashion. (Virtual Machine is abbreviated VM and a VM on AWS is further
jargonized to ***EC2 instance***. EC2 stands for Elastic Cloud Compute; and that is what AWS calls their 
VMs.)


If one were to leave a powerful cloud machine running over 
the weekend (nobody stopped it): That will burn $60 of the cloud computing budget for no reason. 
Our goal here will be to start each instance at 7AM Pacific Time and stop it at 6PM, Monday through Friday. 


If a researcher wanted to use this VM on the weekend the simplest thing approach is to log in to the 
AWS console, navigate to the list of VMs, and *Start* the one they wanted. 


# Synopsis


AWS compute infrastructure is secure; and this means some layers of abstraction are necessary. So what we are building
here can appear to be a bit *elaborate*. Here is the medium-level overview of the procedure.


# Thorough Walkthrough

## Get started

> Suggestion: Read through these notes, then go through the procedural at the above link.

- Log on to the AWS console as an IAM user with *admin* privileges
- Follow this 
[documentation link](https://aws.amazon.com/premiumsupport/knowledge-center/start-stop-lambda-cloudwatch/)
    - This page is a more detailed "personal narrative" of running through that documentation
 

A Lambda function is serverless code that runs on AWS in response to a trigger of some sort. 
We will build **Start** and **Stop** Lambda functions that will be triggered by a daily CloudWatch
alarm.

> ***IMPORTANT WARNING:*** If you already have an EC2 **work instance** running that includes work you value: 
> *Do Not Wire These Lambda Functions Into That Instance!!!*  Something could go wrong. Instead set up
> a test EC2 instance and only when you are satisfied everything is working properly should you re-point
> the Lambda functions at your work instance. 
>
> To be super-safe do these two things with your valuable EC2 work instance:
> - Turn on Terminate protection for the instance. This makes it harder to delete.
> - Save the work instance as an Amazon Machine Image (AMI) (allows you to reconstitute your instance should it accidentally be terminated)


## Starting an EC2


Here are notes on all seven console-wizard start steps for Launching an EC2.


* Step 0 Region: Before starting the Launch wizard: Select a nearby Region (upper right), in my case US-West-2 Oregon
* Step 1 Image (OS): Ubuntu Server 64-bit x86; noting this is a choice of an AMI
    * The Quick Start is one of four tabs, the others being My AMIs, AWS Marketplace and Community AMIs
    * Question for Joel: Talk about these other tabs, how they might jump start a project
* Step 2 Instance Type: **c4.large** as a cheap test VM
* Step 3 Configure Instance... here we use strictly Default values; and I add notes on "What is this option good for?"
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
* Step 4 Add Storage
    * Upped my may file system to 96GB
        * For Joel: Adding additional EBS volumes raises questions
        * Why add new volumes rather than just make the main volume large?
        * Are there Snapshot differences?
        * What is the performance hit for chosing encryption? Sub-options? Cost?
        * ...and so on: There is a lot to unpack here
* Step 5 Tags
    * Notice the inheritance of tags across VMs, volumes and network interfaces: Nice feature
    * For Joel: What sort of conventional wisdom exists for tagging practice?
* Step 6 Configure Security Group
    * Create new; name and description are defaults
        * For Joel: How should we think about SGs? 
        * Selecting an existing one raises the question: Who or what if anything is using this SG?
        * It would seem I have a vast army of Zombie SGs
    * Type SSH
    * Protocol TCP
    * Port Range 22
    * Source Description Custom 0.0.0.0/0
    * Description: Default text
    * Add Rule: No rules added
        * For Joel: Let's review all of these!
        * Edit the SG on the fly? Or after launch?
* Step 7 Review
    * No changes
    * Click Launch
    * Pop up: Generate new PEM file, download: This will need to be permission-modified `chmod 400 file.pem`
    * Launch

The instance started up; so I immediately stopped it via the console. I will continue work later so I will re-start it then.


## Policy and Role

A **policy** on AWS is a JSON-format document that contains some logical language about what some *entity* can do. In 
this case the entity will be able to start and stop EC2 instances in my account. A **role** on AWS is an IAM label. It can
be affixed to an entity like (in this case) a Lambda function. The **role** has a **policy**; so this role/policy business
is the AWS abstraction that permits us to build secure infrastructure. 


Per the documentation I now create a policy and a role prior to creating the Lambda functions for start and stop. 
The Lambda functions will be assigned the role and thereby inherit the policy so the system allows them to start and 
stop instances. One more note: Policies exist globally on AWS; they are not confined to Regions.


AWS manages a long list of policies that are available for adoption by us Users. From the IAM Dashboard just the 
policies we have conscripted for use are linked to. 


* Question for Joel: Customer managed is mutually exclusive to AWS managed? What is third category 'AWS Managed job function'? 
    * Degeneracy in Customer managed (e.g. basic Lambda execution): Can we clean this up? How to proceed?


* Create policy
    * Visual editor tab: You would think "Set Service to Lambda" but in fact: Do nothing. Switch to the JSON tab.
    * JSON tab: Paste the policy text from the documentation
    * Visual editor tab: Go back here and notice that the pasted JSON text has created two elements
        * CloudWatch Logs (3 actions)
        * EC2 (4 actions)
            * For Joel: 3 Resource warnings and etcetera: How much do we need to learn about what goes on here? 
    * Tags: Added `Name` as `ec2-start-stop`
    * Review: Added Name (same as tag)
    * Create Policy: ok


* Create Role
    * Select **Lambda** --> Next: Permissions
    * Add the above Policy
        * Often at this stage one might attach *multiple* policies. 
            * The console keeps track of what we have selected (the full list may not be visible)
            * We double-check at the end
    * Tags: Added Name, Owner, Zombie remarks as above
    * Review: Role name will be **ec2-start-stop**; ensure the Policy is the one intended
        * Also add a second policy to this Role: **AmazonSNSFullAccess**
            * This allows us to send notification emails from the Lambda functions when they run
    * Create role


## Lambda functions


We are creating two Lambda functions: **ec2-start** and **ec2-stop**. Since I did **start** first I made sure 
the EC2 test instance was *stopped*. 


> Preferred practice: Create the Lambda functions in the same Region as the EC2 instance(s) of interest. 


* Lambda console -> Create function -> Author from scratch
    * Basic info: 
        * Name: **ec2-start** / **ec2-stop**
        * Runtime: Documentation suggest Python 3.8; however I am choosing Python 3.9
        * Permissions: Documentation a little out of date here; main thing is to be sure to select the Role created above
        * **Create function** now takes us to the main Lambda configuration console

    * Lambda configuration console
        * Note there are six tabs below the Overview pane: Code, Test, Monitor, Configuration, Aliases, Versions
        * Choose the Code tab
            * In the code source window: click the left sidebar `lambda_function.py` to activate the editor for this file
            * Edit the file and click **Deploy** to make sure it registers
        * Choose the Configuration tab
            * Select the environment variables sub-tab (left side) and add the environment variables and values
        * Test the Lambda function from the Test tab: Click **Test**.
            * When the two Lambdas run they should respectively start / stop the test EC2 instance

> This is the **start** code. The **stop** code: Just make the obvious changes to the last three lines.


```
import json, os, boto3, datetime

region = os.environ['region']
sns_topic = os.environ['sns_topic']
friendly_EC2_name = os.environ['friendly_EC2_name']
friendly_project_name = os.environ['friendly_project_name']
instance_id = os.environ['instance_id']                       # something like 'i-aaabbbcccdddeeeff'

ec2 = boto3.client('ec2', region_name=region)
instances = [instance_id]

def lambda_handler(event, context):
    print('start starting instances')
    ec2.start_instances(InstanceIds=instances)
    print('done starting instances')                         # want to replace with an SNS email message!
```

**Stop** Lambda code last three lines become: 

```
    print('start stopping instances')
    ec2.stop_instances(InstanceIds=instances)
    print('done stopping instances') 
```


The environment variables allow us to keep inside information out of the Lambda code; so it can safely
be published on GitHub.


At the moment there is a bit of extra machinery in this code. This will be built out later so we can send a 
notification email. 


What is missing is the automated timer -- an alarm clock -- that goes off every day to trigger these
Lambda functions. 


## Creating the EventBridge alarm trigger

* Choose + Add trigger --> EventBridge
* Name, description, choose `Schedule` and `Expression`
* For start time: `cron(0 14 ? * MON-FRI *)` translates as 2PM Zulu every week-day (7AM Pacific)
* For stop time: `cron(0 1 ? * TUE-SAT *)` translates as "1AM Zulu every Tuesday through Saturday" (10AM Pacific)
    * In the third field the ? wildcard means use days of week, not days of month
* Return to the Lambda function and refresh: The EventBridge alarm appears as a trigger


## Creating an email notification

- Make the Lambda able to publish via policy/roll
- Expand out the code as shown below
- Create an SNS topic (not `.fifo`) that is reflected in the environment variable
    - Add subscribers, e.g. choosing email and giving email addresses. Recipient must confirm.
- Make sure the start and stop Lambdas send the appropriate messages


Here is the Lambda ec2-stop code:


```
import json, os, boto3, datetime

region = os.environ['region']
sns_topic = os.environ['sns_topic']
friendly_EC2_name = os.environ['friendly_EC2_name']
friendly_project_name = os.environ['friendly_project_name']
instance_id = os.environ['instance_id']                       # something like 'i-aaabbbcccdddeeeff'
account_number = os.environ['account_number']

ec2 = boto3.client('ec2', region_name=region)
instances = [instance_id]

def lambda_handler(event, context):
    print('start stopping instances')
    ec2.stop_instances(InstanceIds=instances)
    print('done stopping instances') 
    
    email_body    = 'stopped EC2 ' + friendly_EC2_name + ' in project ' + friendly_project_name + '\n\n\n'
    email_subject = 'EC2 stop'
    sns           = boto3.client('sns')
    arnstring     = 'arn:aws:sns:' + region + ':' + account_number + ':' + sns_topic
    response      = sns.publish(TopicArn=arnstring, Message=email_body, Subject=email_subject)
```


Here is the Lambda ec2-start code: 


```
import json, os, boto3, datetime

region = os.environ['region']
sns_topic = os.environ['sns_topic']
friendly_EC2_name = os.environ['friendly_EC2_name']
friendly_project_name = os.environ['friendly_project_name']
instance_id = os.environ['instance_id']                       # something like 'i-aaabbbcccdddeeeff'
account_number = os.environ['account_number']

ec2 = boto3.client('ec2', region_name=region)
instances = [instance_id]

def lambda_handler(event, context):
    print('start starting instances')                        # gets written to log file
    ec2.start_instances(InstanceIds=instances)
    print('done starting instances')                         # want to replace with an SNS email message!

    email_body    = 'started EC2 ' + friendly_EC2_name + ' in project ' + friendly_project_name + '\n\n\n'
    email_subject = 'EC2 start'
    sns           = boto3.client('sns')
    arnstring     = 'arn:aws:sns:' + region + ':' + account_number + ':' + sns_topic
    response      = sns.publish(TopicArn=arnstring, Message=email_body, Subject=email_subject)
```

