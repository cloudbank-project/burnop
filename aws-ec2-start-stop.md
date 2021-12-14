> This is a working document as the subject matter is a launching point for further "Researcher-built" IT infrastructure
> on the AWS cloud. At the moment there are a lot of references to someone named "Joel" concerning questions we would 
> like to address. It's a bit like a Beckett play: Joel will be here at any moment. Main thing: This document and the 
> [AWS document it refers to](https://aws.amazon.com/premiumsupport/knowledge-center/start-stop-lambda-cloudwatch/)
> can save you a phenomenal amount of money on cloud use. The time cost is a few hours learning the basic machinery of 
> Amazon Web Services. 


# Introduction


These are notes for managing cost on AWS by turning EC2 Virtual Machines on and off automatically. 
In setting it up we also enable manual on / off by means of an AWS API Gateway service.
We proceed in three 
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


Jargon so far: 


* Serverless function on AWS: **Lambda function** (includes a few lines of Python code in this case)
* Badge affixed to a Lambda: Is called a **Role**
* Logical permission construct, as a text (typically JSON) document: Is called a **Policy** and it is attached to a **Role**
* Virtual Machine where you, the Researcher, do your work: On AWS is called an **EC2 instance**


If one were to leave a powerful cloud machine running over 
the weekend (nobody stopped it): That will burn $60 of the cloud computing budget for no reason. 
Our goal here will be to start each instance at 7AM Pacific Time and stop it at 6PM, Monday through Friday. 


If a researcher wanted to use this VM on the weekend: Uh oh, the VM (the EC2 instance) is *stopped*. How to
start it? One approach is to log in to the AWS console, navigate to the list of VMs, and *Start* the 
one we want. In the walk-through below we provide a simpler method that uses a URL to flip the switch.


Methods to turn EC2 instances on / off manually:


* Establish a **Lambda function** triggered via API Gateway; and use two URLs for **start** / **stop**
* Log in to the AWS console through a browser, navigate to the EC2 dashboard, **start** / **stop** the instance
* Install a command line utility on a local machine and issue a **start** / **stop** command using that utility
* Install an AWS-interact library called **boto3** and use Python code to **start** / **stop** the VM


# Synopsis


AWS compute infrastructure is secure; and this means some layers of abstraction are necessary. So what we are building
here can appear to be a bit *elaborate*, like a Rube Goldberg machine. Here is the medium-level overview of the procedure.


Let's start with the abstract notion of an **alarm clock**. This comes from an AWS service called **Event Bridge**. 
So we set two alarms using `cron` time notation. One goes off every day at 7AM Pacific time. The other goes off 
at 6PM. They are respectively attached as triggers to two AWS *serverless* Lambda functions **ec2-start** and **ec2-stop**.


The Lambda functions have a small block of Python code available to run when the Lambda is triggered by these alarms. 
Again: Lambda function**s** plural because there is one to stop the EC2 instance and one to start it. 
The code is almost entirely the same between them and breaks down into four sections: 


- Import a couple of key libraries
- Obtain a few environment variables from a local secure *Configuration* structure
- Make a Python call to the AWS API to stop/start a test instance
- Send an email reporting this action (for verification purposes)


For these Lambda functions to operate we need two more entities in play on AWS. First we must have an actual 
EC2 instance to start and stop. This has a unique ID, a string that is retained as one of those *Configuraion* values.
In passing: Keeping identification strings safely hidden away, separate from Lambda code, is a good security practice.


The second thing we need is an abstraction called a **Role** that has two **Policies** attached. This **Role** can be 
conveyed upon each of the Lambda functions; rather like a badge. The two **Policies** enable any entity to (a) start or 
stope EC2 instances and (b) use the AWS Simple Notification Service to send an email message. 


So now the sequence of events runs as follows: 


- The EventBridge alarm goes off at 7AM Pacific time
- The ec2-start Lambda code runs
    - It imports necessary libraries and then reads some securely stored configuration values
    - It starts the EC2 instance 
        - This is possible only because the Lambda function has a Role with a Policy that permits this action
    - It sends an email reporting this action to whoever is on the associated SNS topic email distribution list
        - This is possible only because the Lambda function has a Role with a Policy that permits this action
- The EC2 instance -- now running -- is available for use
- Later in the day, at 6 PM Pacific time, the same thing happens again only this time to *stop* the EC2 instance
    - Eventually once we are tired of all the email notifications we might turn them off


Because the EC2 instance is not running from 6PM to 7AM it costs 11/24 as much as an instance that is on all the time.
Of course this is for only 5 of 7 days in the week. Over the weekend the instance is stopped and is not 
accruing further cost. 


Finally just because the Lambda function has a trigger (the EventBridge alarm clock), that does not mean it can't 
have a second trigger as well. We will add this trigger in the form of the AWS API Gateway service. Setting this
up is very fast, a matter of a couple of minutes. The result of the creation is a URL like this one:


**```
https://alphabetsoup.execute-api.us-west-2.amazonaws.com/default/ec2-start
```**


Simply entering this URL into a browser address bar is sufficient to trigger the Lambda function and start the 
instance.


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
* For Joel: Once it exists; shall we set up a separate Lambda that can notice a signal and start it?


The instance started up; so I immediately stopped it via the console. I will continue work later so I will re-start it then.


## Policy and Role

A **policy** on AWS is a JSON-format document that contains some logical language about what some *entity* can do. In 
this case the entity will be able to start and stop EC2 instances in my account. A **role** on AWS is an IAM label; 
like a badge that a sheriff might wear in a western film. A **role** can
be affixed to an entity like (in this case) a Lambda function. The **role** has a **policy**; so the *role/policy* business
is the AWS abstraction that permits us to build secure infrastructure. If your entity is designed to perform 
an operation on AWS: It can't do that operation without first taking on a role with an appropriate policy or policies attached.


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


> This is the **start** code. The **stop** code: Just make the obvious changes to the three key lines of code.


```
import json, os, boto3, datetime

# relates to sending the email notification via SNS
region = os.environ['region']
sns_topic = os.environ['sns_topic']
friendly_EC2_name = os.environ['friendly_EC2_name']
friendly_project_name = os.environ['friendly_project_name']
account_number = os.environ['account_number']

# relates to starting the instance
instance_id = os.environ['instance_id']                      # something like 'i-aaabbbcccdddeeeff'
instances = [instance_id]
ec2 = boto3.client('ec2', region_name=region)

def lambda_handler(event, context):
    
    print('start starting instances')                        # print() writes to log file
    ec2.start_instances(InstanceIds=instances)
    print('done starting instances')

    sns           = boto3.client('sns')
    email_body    = 'started EC2 ' + friendly_EC2_name + ' in project ' + friendly_project_name + '\n\n\n'
    email_subject = 'EC2 start'
    arnstring     = 'arn:aws:sns:' + region + ':' + account_number + ':' + sns_topic
    response      = sns.publish(TopicArn=arnstring, Message=email_body, Subject=email_subject)
```

**Stop** Lambda code: The three key 'start' lines become: 

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


## Creating an API Gateway trigger


Suppose we also wish to be able to turn an EC2 instance on / off using the same two Lambda functions. 
This can be done automatically every day as described below using an EventBridge alarm trigger. The 
alarm "goes off" and triggers the Lambda function to run. We can also add an API Gateway trigger that
allows us to turn the instance on / off using an endpoint URL. 


- Choose + Add trigger --> API Gateway
- Dropdown for "Create / attach existing" --> choose Create an API
- API type: HTTP API
- Security: Open
    - For Joel: Evaluate alternatives and risk
- Other fields: Default values are fine
- Click the **Add** button


This will produce an endpoint URL found in the Lambda function page under the Configuration tab. 
Pasting this URL into a browser will run the Lambda function and start or stop the instance.
It also brings up the text response `null` in the browser when it runs.



## Creating the EventBridge alarm trigger

* Choose + Add trigger --> EventBridge
* Name, description, choose `Schedule` and `Expression`
* For start time: `cron(0 16 ? * MON-FRI *)` translates as 4 PM Zulu every week-day (8 AM Pacific Standard)
* For stop time: `cron(0 2 ? * TUE-SAT *)` translates as "2AM Zulu every Tuesday through Saturday" (6 PM Pacific Standard)
    * In the third field the ? wildcard means use days of week, not days of month
* Return to the Lambda function and refresh: The EventBridge alarm appears as a trigger


> Notice that these alarms will trigger a stop of the VM at 6PM; and this will happen regardless of whether 
> a researcher is logged in, editing a file, hasn't saved their work for three hours... so "*save early save often*"


## Creating an email notification

- Create an SNS topic (not `.fifo`)
    - It's **Name** is the same as the `sns_topic` environment variable
    - Add subscribers to the topic: Choose email and provide email addresses. Recipients must confirm.
- Make sure the Lambda function's **Role** includes the policy **AmazonSNSFullAccess** as noted above
- The code below uses the environment variable `account_number`. This is the 12-digit AWS account number (no hyphens).
- The code below is as above with the SNS email notification added
    - This requries five additional lines of code



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

