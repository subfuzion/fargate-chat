## Fargate Chat Demo

This fork is based on work by Ric Harvey.  
Twitter: [@ric__harvey](https://twitter.com/ric__harvey)  
Repo: https://gitlab.com/ric_harvey/bl_practical_fargate.git

This is a working sample that deploys a simple chat application using AWS Fargate. The sample includes creating a VPC and Fargate cluster, using service discovery, and deploying containerized services to the cluster. VPC and an AWS Fargate cluster. Into this cluster you'll learn how to deploy a container service whilst using service discovery. 

### What is Fargate

URL: [https://aws.amazon.com/fargate/](https://aws.amazon.com/fargate/)

AWS Fargate is a technology for Amazon ECS that allows you to run containers without having to manage servers or clusters. With AWS Fargate, you no longer have to provision, configure, and scale clusters of virtual machines to run containers. This removes the need to choose server types, decide when to scale your clusters, or optimize cluster packing. AWS Fargate removes the need for you to interact with or think about servers or clusters. Fargate lets you focus on designing and building your applications instead of managing the infrastructure that runs them.

Another bonus to Fargate is that apart from no infrastructure to manage you are charged only for the resources your container consumes and not a full EC2 instance.

Checkout: [https://aws.amazon.com/fargate/pricing/](https://aws.amazon.com/fargate/pricing/) for more details

### Requirements

- AWS account
- AWS CLI installed + configured with your API keys
- Git CLI installed

### Expected time

You should be able to deploy this in 30-45 mins

### Solution overview

We are going to use the AWS technologies listed below to build a simple chat application that uses redis to pass messages between the frontends. When we build the components we'll use CloudFormation however when we deploy the redis solution we'll build the "service" definition by hand in order to highlight how you can utilise the service discovery.

Products used:

- AWS Fargate
- Amazon CloudFormation
- AWS Application Discovery Service

##### High level overview diagram

![High level Diagram](./img/chat-app.png)

## Exercise

### Getting started

##### 1. Build the VPC

First we are going to need a VPC into which we'll deploy our Fargate Cluster. We are going to use CloudFormation and deploy three public subnets and three private subnets, our fargate cluster will be in the private subnet and we'll use load balancers to make this publicly accessible.

In this git repository there is a folder called `cloudformation`. This contains our files for building our VPC.

**Using the AWS console:**

Open the AWS console in your browser and go to CloudFormation. Press Create Stack.

![New Stack](./img/newstack.png)

Select upload template to S3 and find the file named **_public-private-vpc.yml_**

![Template](./img/template.png)

Answer a few questions. For **Stack name** use "chat-vpc" and for **EnvironmentName** use a common name that will be used when deploying the other templates as well, such as "chat". Accept default values for everything else.

Wait for this to complete in the console.

**Using the AWS CLI:***

    $ aws cloudformation deploy --template-file ./cloudformation/public-private-vpc.yml --stack-name chat-vpc --parameter-overrides EnvironmentName=chat

##### 2. Create a Fargate Clusterj 

We now need to create our Fargate cluster namespace. Note there are no running instances in EC2 but we still have a cluster in the ECS console. Our CloudFormation stack is also going to create a public application load balancer.

Press Create Stack again and upload the **fargate-cluster.yml** file, or enter the following for the AWS CLI:

    $ aws cloudformation deploy --template-file ./cloudformation/fargate-cluster.yml --stack-name chat-cluster --capabilities CAPABILITY_IAM --parameter-overrides EnvironmentName=chat

For **Stack name** use "chat-cluster" and remember to use the same **EnvironmentName** value as before ("chat").

Once complete head over to the ECS console and you'll see your newly created cluster.

![Stacks](./img/stacks.png)

### 3. Create the Redis lead task and service definition

Let's now create a redis container in our fargate cluster. Open the CloudFormation console and load the **redis-lead.yml** template. Or, using the AWS CLI:

    $ aws cloudformation deploy --template-file ./cloudformation/redis-lead.yml --stack-name chat-redis --parameter-overrides EnvironmentName=chat

Look in the ECS console under Task Definitions and you should now see a new task called **redis-lead**. Check the tick box under actions and choose **Create Service**.

![Create Stack](./img/create-service.png)

Now set the inital options for the service:

![Set variables for the stack](./img/config-service.png)

Choose:

- **Launch Type**: Fargate
- **Number of Tasks**: 1

The rest can remain default. The next section will ask you about networking configuration:

![Redis networking](./img/network-service.png)

Make sure you select the following options:

- **Cluster VPC**: chat-vpc (subnet should be 10.0.0.0/16)
- **Subnets**: 10.0.3.0/24, 10.0.4.0/24

and next we need to edit the security group:

![security group](./img/security-service.png)

Change the type from **_HTTP_** to **_custom TCP_** and set the port to 6379. I'm now going to lock redis down so only hosts in my network can access it by setting the source to 10.0.0.0/16. I recommend that if you do this in production you can go even further and create tighter security group rules.

Set:

- **_Auto-assign public IP_**: disabled

Now for some service discovery so our chat application can locate the redis lead, via a DNS name rather than hardcoding an IP.

![service discovery](./img/discovery-service.png)

The defaults are good to go. What this will do is create a .local DNS zone in Amazon Route 53

The only change you need to make is to drop the TTL for the A record to at most 3 seconds. When setting this value consider, if that service was to fail and be replaced with a container with a new IP the TTL is downtime till the new version propagates.

![DNS TTL](./img/dns-service.png)

Click Next, next, create service.

Now the service is created, you can check out the status of the running container in the ECS console. Its also work noting if you look in EC2 you wont see any running instances. To check out the DNS do this by going to Route 53 and looking for the **_local_** zone. In here you should see the record **_redis-lead.local._** which you'll see has a private IP in one of the ranges 10.0.{3-5}.0/24

### Create the chat application

Finally lets create the application. In CloudFormation load the **chat-service.yml** template, or use the AWS CLI:

    $ aws cloudformation deploy --template-file ./cloudformation/chat-service.yml --stack-name chat-service --parameter-overrides EnvironmentName=chat
    
This stack creates the task definition and service this time so all you have to do make sure you select the right environment. We will create 3 copies of our chat application (3 tasks) and link it to the public application load balancer (ALB). If you check out the template you'll notice there is a section to give the ALB a nice public DNS name; if you have a domain name you want to use, you'll need to uncomment and update the section appropriately.

**NOTE:** Give a few mins for the DNS to propagate before trying to browse to it.

Once the stack completes, head to the ECS console and you'll see in your cluster you have 2 services and 4 running tasks. Thats because the redis service has 1 copy and the chat-service has 3 copies of the task.

### Testing

Load your favourite web browser and find the FQDN for the public ALB created. This can be found in the EC2 console under load balancers. Head to the URL and type in your name, you can then start chatting.

![Chat app screenshot 1](./img/screenshot1.png)
![Chat app screenshot 2](./img/screenshot2.png)

### Summary

In summary, you've now successfully built a Fargate cluster, loaded a task definition and built a service by hand. For that service you've also enabled service discovery so our chat service can use the redis instance. Plus you've done all this without spinning up and configuring any servers!

### Clean up

Most of this can be deleted by deleting the stack in the CloudFormation console. However you'll need to manually pull down the service discovery component as we built this by hand.

First lets delete the service from in the ECS console. Find the redis lead service in your cluster and select it then hit delete:

![find the service](./img/delete-service.png)

Now you can confirm deletion by typing **_delete me_**

Lets now use the aws CLI to clean up the service discovery elements. Let's find the service and namespace we need to clean up.

```bash
$ aws servicediscovery list-namespaces

{
    "Namespaces": [
        {
            "Id": "ns-orjxhzxsj7yk7ocm",
            "Arn": "arn:aws:servicediscovery:eu-west-1:210944566071:namespace/ns-orjxhzxsj7yk7ocm",
            "Name": "local",
            "Type": "DNS_PRIVATE"
        }
    ]
}
```

```bash
$ aws servicediscovery list-services

{
    "Services": [
        {
            "Id": "srv-a66t7vfsvb37jnhi",
            "Arn": "arn:aws:servicediscovery:eu-west-1:210944566071:service/srv-a66t7vfsvb37jnhi",
            "Name": "redis-lead"
        }
    ]
}
```

Then delete the service first and lastly the namespace:

```bash
$ aws servicediscovery delete-service --id srv-a66t7vfsvb37jnhi
$ aws servicediscovery delete-namespace --id ns-orjxhzxsj7yk7ocm
```

### What next?

Some ideas:

If you want to take this to the next level, try creating a clustered redis service or making the chat front end autoscale. Another good thing to experiment with is adding IAM roles to your tasks.
