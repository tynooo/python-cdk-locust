

# Deploy a Locust load generator in ECS using CDK Python 

This lab walks you through creating a CDK project in Python that will implement 
an ECS Service running [Locust.io](locust.io). Using CDK constructs 
we'll create a customised Locust container image, and all supporting service configuration, including: VPC, 
ECS cluster, ECS Service, Application Load Balancer, and a CloudWatch dashboard.

## Getting Started

Go to Cloud9 on the AWS Web console - this is your IDE in the cloud. 

On top right corner - Switch to the correct region
on the left hand pane click on Your environments
Open the python_cdk_locust environment in Cloud9

![Cloud9 Environments](images/Cloud9_Envs.png)

## Step 1: Initialise a CDK project

**todo:** 
1. How will these be hosted? 
2. Do we neeed to install CDK?
3. Add git clone to grab dependecies (dockerfile and locust.py)

**/todo**


Open a terminal tab in Cloud9 by clicking the + and selecting New Terminal

![Cloud9 New Terminal](images/Cloud9_Term.png)

1. Enter the lab directory and create a new directory for your work 
```
cd python-cdk-locust-lab
mkdir lab
cd lab
```
2. Initialise your CDK project
```
cdk init --language python
```
This builds the base directory structure for your project, it must be run in an
empty directly or it will fail. Some important files:
 * app.py - The entry point for you app, it defines your environment  and the 
stack(s) that will be created
 * requirements.txt - Library dependencies for your Python code, in this case 
the CDK libraries that we will use
 * lab/lab.py - The file that defines the CDK stack

 
## Step 2: Set the region that our stack will deploy into
Open the app.py file in Cloud9, and add an environment paramater to the stack
instantiation. If you're deploying into a different account, you can set that 
here too
```
PythonCdkLocustLabStack(app, "python-cdk-locust-lab",
    env={'region': 'ap-southeast-2'}
)
```
**Remember to save your files after each step!**

## Step 3: Define your python dependencies
Using Cloud9, create a file named requirements.txt in your lab directory, add 
the following, and save it. This file defines the Python dependencies that our 
lab will use. 
```
aws-cdk.core
aws-cdk.aws_ec2
aws-cdk.aws_ecs
aws-cdk.aws_cloudwatch
```
Open your CDK Stack lab/lab.py in Cloud9 and replace the first line with the following
to import the CDK libraries.

```
from aws_cdk import (
    core,
    aws_ec2 as ec2,
    aws_ecs as ecs,
    aws_ecs_patterns as ecs_patterns,
    aws_cloudwatch as cw
)
```


# Let's Build!

In CDK there are 3 levels of construct:
1. CFn Resources - These are constructs which are created directly from 
CloudFormation and work in the same way as the CloudFrmation resource 
they're based upon, requiring you to explicitly configure all resource 
properties, which requires a complete understanding of the details of the 
underlying resource model.
2. AWS constructs - The next level of constructs also represent AWS resources, 
but with a higher-level, intent-based API. AWS Constructs offer convenient 
defaults and reduce the need to know all the details about the AWS resources they represent.
3. Patterns - These constructs are designed to help you complete common tasks in 
AWS, often involving multiple kinds of resources.

You can find more information on CDK constructs in the CDK Developer Guide - 
https://docs.aws.amazon.com/cdk/latest/guide/constructs.html

In this lab we'll be using a Pattern from the ecs_patterns class
called ApplicationLoadBalancedEc2Service. However, we'll create the ECS cluster
independently of the Pattern.


## Step 4: Create an ECS Cluster

Create an ECS cluster and add an instance to it. If we had specific requirements
around the VPC configuration, we could have created a fresh one first, and passed
it to the ECS cluster via the ```vpc``` parameter. Instead, we'll just let the 
ECS construct create it for us.


```
        loadgen_cluster = ecs.Cluster(
            self, "Loadgen-Cluster",
        )
        
        loadgen_cluster.add_capacity("Asg",
            desired_capacity=1,
            instance_type=ec2.InstanceType("t3.large"),
        )
        
```

**todo: Add a screenshot**

As you progress, you can test that your code generates valid CloudFormation by 
running ```cdk synth``` But before you do this, you need to activate your python 
virtualenv and install your dependencies. 

```
source .env/bin/activate
pip install -r requirements.txt
```

***Every time you work on your CDK project, you'll need to activate your virtualenv***

Now run ``` cdk synth``` to synthesize your CloudFormation template. You should 
see a CloudFormation template defining your ECS cluster and its dependecies


```
(.env) Admin:~/environment/python-cdk-locust-lab/lab (master) $ cdk synth
Resources:
  loadgenvpc044E9081:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.7.1.0/24
      EnableDnsHostnames: true
      EnableDnsSupport: true
      InstanceTenancy: default
      Tags:
        - Key: Name
          Value: python-cdk-locust-lab/loadgenvpc
    Metadata:
      aws:cdk:path: python-cdk-locust-lab/loadgenvpc/Resource
...
```

You can now deploy your template by running ```cdk deploy``` 


**todo:Change to the right path and screenshot!**

## Step 5: Create a container and task definition

Now we've got our ECS cluster, we need something to run on it. We'll start by 
defining a task definition
```
        task_def = ecs.Ec2TaskDefinition(self, "locustTask",
            network_mode=ecs.NetworkMode.AWS_VPC
        )
```

Now we'll add a container to run in our task. This container definition creates 
a new container image based on a DOCKERFILE which references the official Locust.io 
image on Dockerhub and accompanying locust.py file in the ```/locust``` directory. 
CDK will then upload this to an ECR repository that it creates automatically.

We also set the environment variables that Locust requires to initialise here.

```
        locust_container = task_def.add_container(
            "locustContainer",
            image=ecs.ContainerImage.from_asset("~/environment/python-cdk-locust-lab/locust"),
            memory_reservation_mib=512,
            essential=True,
            logging=ecs.LogDrivers.aws_logs(stream_prefix="cdkLocust"),
            environment={"TARGET_URL": "127.0.0.1"}
        )
        locust_container.add_port_mappings(ecs.PortMapping(container_port=8089))
```


## Step 6: Define a service

Now that we've created all of the underlying components, it's time to put them
together into a service and run it on our ECS cluster.
```
        locust_service = ecs.Ec2Service(
            self, "locustService",
            cluster = loadgen_cluster,
            task_definition = task_def,
            security_group = security_group,
            desired_count = 1
        )
```


As we're using a Pattern, CDK takes care of the rest of the details. 

Run ```cdk diff``` to see what changes this will make to the CloudFormation that CDK
will generate and deploy.

Now deploy your stack using ```cdk deploy``` and when that completes, go to the 
LocustServiceURL which is output at the end of the deploy process in a web 
browser. You should see your Locust load generator page

## Oops! We need a bigger instance!

Imagine having just discovered that the t3.large instance we selected for our ECS
cluster isn't adequate to handle the load we're generating. We need to move to a
C5.large instance.

With CDK, this is easy. Simply change the ```t3.large``` in your ecs cluster to
```c5.large``` save the file, and redeploy.

## Monitoring
Obviously, just having our service running is not enough, we need to monitor its
operation. Let's create a simple CloudWatch Dashboard to monitor a couple of 
metrics for our service. 


## Cleanup

Cleaning up in CDK is easy! Simply run ```cdk destroy``` and CDK will delete 
all deployed resources
