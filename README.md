# burnop: A CloudBank Solution

This Cloudbank Solution helps you (a research team using the cloud) ***optimize*** your use (***burn***) of cloud resources.

The term **burn** in this context we mean *"like rocket fuel"*. You have a cloud budget and you have 
computations to perform on your data. As you compute: How fast are you burning through your budget? 


CloudBank has some positive news on this topic: Even if you are not a cloud computing expert there are some steps you 
can follow to help you spend less on your computations. The down side is it may take time and effort; so we encourage
you to evaluate the tradeoffs.


## The Basics

* Code profiling
* Checkpointing
    * Pre-emptible instances ([AWS](https://aws.amazon.com/ec2/spot/)) ([GCP](https://cloud.google.com/preemptible-vms/)) ([Azure](https://docs.microsoft.com/en-us/azure/batch/batch-low-pri-vms))
* Monitoring tools: Top, dashboard monitors


## The Jargon

* vCPUs
* cores
* threads
