# Serverless Inference with MMS on FARGATE

This is self-contained step by step guide that shows how to create ECS with Fargate in order to do a serverless inference with MMS. 

## Prerequisites

Even though it is fully self-contained we do expect reader to have some knowledge about the following topics:

* [MMS](https://github.com/awslabs/mxnet-model-server)
* [What is Amazon Elastic Container Service (ECS)](https://aws.amazon.com/ecs)
* [What is Fargate](https://aws.amazon.com/fargate)
* [What is Docker](https://www.docker.com/) and how to use containers

Since we are doing inference, we need to have a pre-trained model that we can use to run inference. For the sake of this article, we will be using [SqueezeNet model](https://github.com/awslabs/mxnet-model-server/blob/master/docs/model_zoo.md#squeezenet_v1.1). In short, SqueezeNet is a model that allows you to recognize objects on a picture. 

Now, that we have the model chosen let's discuss at a high level what our serverless solution will look like:

![architecture](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/architecture_2.png)

Do not be worried if something is not clear from the picture. We are going to walk you through step by step. And these steps are:

1. Familiarize yourself with MMS containers
2. Create a SqueezeNet task definition (with the docker container of MMS) 
3. Create Fargate ECS cluster
4. Create Application Load Balancer
5. Create Squeezenet Fargate service on the cluster
6. Profit!

Let the show begin...

## Familiarize Yourself With Our Containers 

With the current release of [MMS, 0.3](https://github.com/awslabs/mxnet-model-server/releases/tag/v0.3.0), we now providing official containers are provided with MMS preinstalled. There are 2 containers:

* [awsdeeplearningteam/mms_cpu](https://hub.docker.com/r/awsdeeplearningteam/mms_cpu/)
* [awsdeeplearningteam/mms_gpu](https://hub.docker.com/r/awsdeeplearningteam/mms_gpu/)

In our article we are going to use cpu container (mms_cpu). 

There are several constraints that one should consider when using Fargate:

1. There is no GPU support at the moment.
2. mms_cpu container is optimized for the Skylake Intel processors (that we have on our [C5 EC2 instances](https://aws.amazon.com/ec2/instance-types/c5/)). However, since we are using Fargate, unfortunately there is no guarantee that the actual hardware will be Skylake.

Our containers come with preinstalled config of the SqueezeNet model (such a nice coincidence that we have decided to run inference for exactly this model :) ). Even though the config is pre-baked in the container, it is highly recommended to have a quick look at it. [Familiarize yourself](https://github.com/awslabs/mxnet-model-server/blob/master/docker/mms_app_cpu.conf),since for your own model, most likely you will have to update it. here is the line in the config that is pointing to the binary of the model:

```
https://github.com/awslabs/mxnet-model-server/blob/master/docker/mms_app_cpu.conf#L3
```

Looking closely, one can see that it is just a public HTTPS link to the binary:

```
https://s3.amazonaws.com/model-server/models/squeezenet_v1.1/squeezenet_v1.1.model
```

So there is no need to pre-bake actual binary of the model to the container. You can just specify the HTTPS link to the binary.

The last question that we need to address: how we should be starting our MMS within our container. And the answer is very simple, you just need to set the following [ENTRYPOINT](https://docs.docker.com/engine/reference/builder/#entrypoint): 

```bash
mxnet-model-server start --mms-config /mxnet-model-server/mms_app_cpu.conf
```

And this it, nothing else.

At this point, you are ready to start creating actual task definition.

## Create a SqueezeNet Task Definition

This is the first task where you finally ready to start doing something:

1. Login to the AWS console and go to the Elastic Cloud Service -> Task Definitions and Click “Create new Task Definition”:

![task def](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/1_Create_task_definition.png)

2. Now you need to specify the type of the task, surprise-surprise, you will be using the Fargate task:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/2_Select_Fargate.png)

3. The task requires some configuration, let's look at it step by step. First set the name:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/3_Config_1.png)

Now is important part, you need to create [IAM role](https://aws.amazon.com/iam) that will be used to publish metrics to CloudWatch:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/Task+Execution+IAM+Role+.png)

The containers are optimized for 8 vCPUs, however in this example you are going to use slightly smaller task with 4 vCPUs and 8 GB of RAM:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/cpu+and+ram.png)

2. Now it is time to configure the actual container that the task should be executing.

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/container+step+1.png)

3. The next task is to specify the port mapping. We need to expose container port 8080. 
This is the port that the MMS application inside the container is listening on. 
If needed it can be configured via the config [here](https://github.com/awslabs/mxnet-model-server/blob/master/docker/mms_app_cpu.conf#L40).

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/port+8080.png)

Next, we configure the health-checks. MMS has a pre-configured endpoint `/ping` 
that can be used for health checks.

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/container+health+checks.png)

After configuring the health-checks, we go onto configuring the environment, with the entry point that we have discussed earlier:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/entrypoint.png)

Everything else can be left as default. So feel free to click `Add` to create your very first ECS Fargate-task. If everything is ok, you should now be able to see your task in the list of task definitions.

Finally, you have a task that you can be run. In ECS, Services are created to run Tasks. A service is in charge of 
running multiple tasks and making sure the that required number of tasks are always running, 
restarting un-health tasks, adding more tasks when needed. 
 
 If the service is going to be accessible from the Internet, we first need to configure a load-balancer that will  be 
 in charge of serving the traffic from the Internet and redirecting it to these newly created tasks. 
 Let's create an Application Load Balancer now:

## Create a Load Balancer

AWS supports several different types of Load Balancers:

* Application Load Balancer: works on the level 7 of the OSI model (effectively with the HTTP/HTTPS protocols)
* TCP Load Balancer 

For your cluster you are going to use application load balancer.
1. Open AWS console -> Click Services -> Select EC2
2. Go to the “Load balancers” section
3. Create new Load Balancer

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/1__Create_Load_Balancer.png)

5. Choose “HTTP/HTTPS”

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/2__HTTP_HTTPS+.png)

6. Set all the required details. Most importantly, set the VPC. VPC is very important. 
Having the wrong VPC might cause your LB to not be able to communicate with your tasks. Make a note of the VPC that you 
going to use here for later.

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/3_2_Listeners_and_AZ+.png)

7. Next is configuring the security group. This is also important. Your security group should:

* allow inbound connections for port 80 (since this is the port on which LB will be listening on)
* member of the security group should be able to talk to the security group where you plan to have your service. 

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/4.+Configure+Security+groups.png)

8. Routing configuration is simple. Here you need to create a “target group”. 
But, in your case the ECS service, that you will create later, will automatically create a target group.  
Therefore you will create dummy “target group” that you will delete after the creation of the LB. 

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/5.+Configure+Routing+(DUmmy).png)

9. Nothing needs to be done for the last two steps. `Finish` the creation and ...
10. Now you are ready to remove dummy listener and target group

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/8__Delete_the_dummy_listener.png)

Now that you are `done-done-done` with the Load Balancer creation, lets move onto creating our Serverless inference service.

## Create an ECS Service from the ECS Task Definitions

1. Go to Elastic Container Service → Task Definitions and select the task definitions name. Click on actions and select create service.

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/1.+Go+to+task+definitions.png)

2. There are two important things on the first step (apart from naming):

* Platform version: it should be set to 1.1.0 .
* Number of tasks that the service should maintain as healthy all of the time, in our case we will use 3.

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/number+of+tasks.png)

3. Now it is time to configure the VPC and the security group. You should use the same VPC that was used for the LB (and same subnets!).

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/3.2.1+Use+the+existing+VPC+Edit+sg.png)

4. As for the security group, it should be either the same security group as you had for the LB, or the one that LB has access to:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/3.2.2+SG+Use+existing.png)

5. Now you can connect your service to the LB that was created in the previous section. Select the application load balancer and set the LB name:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/3.2.3+Add+load+balancing.png)

6. Now we need to specify which port on the LB our service should be listening on:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/3.2.4+Configure+load+blancer.png)

7. You are not going to use service discovery now, so uncheck it:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/3.2.5+Next.png)

8. In this document, we are not using auto-scaling options. For an actual production system, it is advisable to have this configuration setup.

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/3.3+Auto+scaling.png)

9. Now you are `done-done-done` creating a running service. You can move to the final chapter of the journey, which is testing the service you created. 

## Test The Inference

First find the DNS name of your LB. It should be in `AWS Console -> Service -> EC2 -> Load Balancers` and click on the LB that you created.

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/lb_dns.png)

Now you can run the health checks using this load-balancer public DNS name, to verify that your newly created service is working:
```bash
curl InfraLb-1624382880.us-east-1.elb.amazonaws.com/ping 
```
```text
http://infralb-1624382880.us-east-1.elb.amazonaws.com/ping
{
    "health": "healthy!"
}
```

And now we are finally ready to run our inference! Let's download an example image:
```bash
curl -O https://s3.amazonaws.com/model-server/inputs/kitten.jpg
```

The image:

![](https://s3.amazonaws.com/mms-github-assets/MMS+with+Fargate+Article/kitten.jpg)

The output of this query would be as follows,

```bash
curl -X POST InfraLb-1624382880.us-east-1.elb.amazonaws.com/squeezenet/predict -F "data=@kitten.jpg"
```

```text
{
      "prediction": [
    [
      {
        "class": "n02124075 Egyptian cat",
        "probability": 0.8515275120735168
      },
      {
        "class": "n02123045 tabby, tabby cat",
        "probability": 0.09674164652824402
      },
      {
        "class": "n02123159 tiger cat",
        "probability": 0.03909163549542427
      },
      {
        "class": "n02128385 leopard, Panthera pardus",
        "probability": 0.006105933338403702
      },
      {
        "class": "n02127052 lynx, catamount",
        "probability": 0.003104303264990449
      }
    ]
  ]
}
```
## Instead of a Conclusion

There are a few things that we have not covered here and which are very useful, such as:

* How to set up IAM policies to cloudwatch metrics.
* How to configure auto-scaling on our ECS cluster.
* Running A/B testing of different versions of the model with the Fargate Deployment concepts.

Each of the above topics require their own articles, so stay tuned !!

## Authors

* Aaron Markham
* Vamshidhar Dantu 
* Viacheslav Kovalevskyi (@b0noi) 