# burnop: A CloudBank Solution

This Cloudbank Solution helps you (a research team using the cloud) ***optimize*** your use (***burn***) of cloud resources.

The term **burn** in this context we means *"like rocket fuel"*. You have a certain cloud computing budget and you have 
computations to perform on your data. If you are not a cloud computing expert: There are some simple steps you can follow
*anyway* that help you get the most compute for your research computing dollar.

## The Basics

* Code profiling
* Checkpointing
    * Pre-emptible instances ([AWS](https://aws.amazon.com/ec2/spot/)) ([GCP](https://cloud.google.com/preemptible-vms/)) ([Azure](https://docs.microsoft.com/en-us/azure/batch/batch-low-pri-vms))
* Top, vCPUs, cores, threads, and dashboard monitors
