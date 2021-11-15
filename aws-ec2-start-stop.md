## Overview


EC2 instances left running (e.g. overnight) cost money. If these instances are doing nothing we stop them and thereby 
stop paying for them. There is 
[documentation](https://aws.amazon.com/premiumsupport/knowledge-center/start-stop-lambda-cloudwatch/)
online for setting this up. This page is essentially a "personal narrative" 
version of those instructions. These notes only cover *Stopping* an EC2 at say 7PM every night. 
Auto-stopping typically saves thousands of dollars in unnecessary cloud spend. 


## Procedural notes

Following the documentation at the link given above... may we


> suggest: Read these notes, then go through the procedural at the above link.


The idea is to build two Lambda functions: One to Start an EC2 and one to Stop an EC2. Call these **Start-Lambda** 
and **Stop-Lambda**. We set a daily CloudWatch alarm for each. When it goes off: The Lambda functions do their job. 
The remaining notes on this page refer only to the **Stop** operation.


> ***WARNING:*** If you already have an EC2 running that includes work you value: Learn to use this
> procedure on a different EC2 in case something goes awry. Only whene you are satisfied (say after
> a couple of days) that it works properly should you re-point it at your working EC2.



* Region: For us at UW the Oregon region is nearest: `us-west-2`. 
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
