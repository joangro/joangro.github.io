---
layout: post
title: Optimizing a Jenkins and Sonarqube instance in GCP - Part I
categories: [Jenkins, Sonarqube, GCP, Cloud, VM]
---

On the latest years, there has been a steady increase on more demanding and more specialized tooling for the day-to-day DevOps tasks. However, there are tools that have proven to stand the test of time, and have evolved to be touchstones on development-centered teams. This is the case of Jenkins, which after so many years is still being used to build and implement modern CI/CD pipelines.

Some other, and slightly more recent tooling, such as [Sonarqube](https://www.sonarqube.org/) is also still used to the date, and it's needed to tackle other specific issues on the DevOps cicle of development and integration. 

This posts focuses on building and showing how these products can be integrated with a modern Cloud platform, in this case, Google Cloud Platform.

## To have into account

- Newer Jenkins versions can use Java 11, however I will use Java 8 on this example.
- Sonarqube requires Java 10+
- Sonarqube requires a BD. I will use postgres for this example.
- Sonarqube is IO intensive.
- Sonarqube requires [some specific parameters](https://docs.sonarqube.org/latest/requirements/requirements/) when running in Linux. I will use a Debian 9 image for this example.

## Setup

### Prerequisites

- Have a [GCP project](https://cloud.google.com/getting-started) up and running.
- Have enabled the [Compute Engine API](https://cloud.google.com/compute/docs/api/getting-started) on that project.
- Have billing enabled for the project. You can use the [GCP free tier](https://cloud.google.com/free), and use the given 300$. It's more than enough to build the resources for this example.
- You can use the [Cloud Shell](https://cloud.google.com/shell) to run the commands for this example.

### Build the infraestructure

Start by exporting some variables to be used for our commands:

```
export ZONE=europe-west1-b
export VM_NAME=jenkins-sonarqube
export NETWORK=default
```

Next, we will build different disks to attach to our instance. 

As mentioned, Sonarqube is intensive on the IO side, so one disk will be dedicated to it. 
The other disk will be used for the Postgres database, also used by Sonarqube.

Create disk for sonarqube:

```
gcloud beta compute disks create $VM_NAME-sonarqube \
	--type=pd-ssd \
	--size=10GB \
	--zone=$ZONE
```

Create disk for Postgres:

```
gcloud beta compute disks create $VM_NAME-db \
	--type=pd-ssd \
	--size=10GB \
	--zone=$ZONE
```

Now, build the VM instance, and attach the created disks to it:


```
gcloud beta compute instances create $VM_NAME \
	--zone=$ZONE \
	--machine-type=n1-highcpu-4 \
	--subnet=$NETWORK \
	--scopes=https://www.googleapis.com/auth/cloud-platform \
	--tags=$VM_NAME \
	--image-family=debian-9 \
	--image-project=debian-cloud \
	--boot-disk-size=10GB \
	--boot-disk-type=pd-standard \
	--boot-disk-device-name=$VM_NAME \
	--disk=name=$VM_NAME-db,device-name=$VM_NAME-db,mode=rw,boot=no \
	--disk=name=$VM_NAME-sonarqube,device-name=$VM_NAME-sonarqube,mode=rw,boot=no
```

**Note** some fields that you might be interested in changing:

- `--machine-type`: I picked a machine with 4 vCPUs and a memory of 3.6 GB. However you can pick any machine type you like from [this list](https://cloud.google.com/compute/docs/machine-types#general_purpose) or even a custom one.
- `--image-family` and `--image-project`: I like Debian, and picked its released version 9 (Stretch). However you can pick any image you like, you can list all of them with the `gcloud compute images list` command.
- For the complete list of the flags used, and their details, you can check [this documentation](https://cloud.google.com/sdk/gcloud/reference/compute/instances/create).

Finally, we will need to create a firewall rule on our used network to SSH into the VM. You can just run the following command:

```
gcloud compute firewall-rules create p22 \
	--allow tcp:22 \
	--direction IN \
	--network $NETWORK \
	--target-tags $VM_NAME \
	--source-ranges 0.0.0.0/0
```

Now, we can SSH into the machine by running this command on the Cloud Shell:

```
gcloud compute ssh $USER@$VM_NAME
```



## Next steps

On Part II we will see how to configure the VM to install the correct Java versions and our CI/CD tools.
