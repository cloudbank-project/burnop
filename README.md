# burnop: A CloudBank Solution

This Cloudbank Solution helps you (a research team using the cloud) optimize your use of (burn of) cloud resources.

When we say **burn** in this context we mean *"like rocket fuel"*. You have a certain cloud computing budget and you have 
computations to perform on your data. If you are not a cloud computing expert: There are some simple steps you can follow
*anyway* that will help you get the most compute for your research computing dollar.

## The Basics

* Code profiling
* Checkpointing
    * Pre-emptible instances ([AWS](https://aws.amazon.com/ec2/spot/)) ([GCP](https://cloud.google.com/preemptible-vms/)) ([Azure](https://docs.microsoft.com/en-us/azure/batch/batch-low-pri-vms)) ([IBM](https://www.ibm.com/cloud/virtual-servers/details) (no low-cost preemptible service visible as of 12/20))
* Top, vCPUs, cores, threads, and dashboard monitors
