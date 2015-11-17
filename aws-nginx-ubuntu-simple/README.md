---


---

## What You Will Build ##

Weave allows you to focus on developing your application, rather than your infrastructure.

In this example we demonstrate how with Weave you can quickly deploy Nginx as
a load balancer for a simple php application running in containers on multiple nodes in [Amazon Web Services](http://aws.amazon.com), with no modifications to the application and also minimal docker knowledge.

![Weave, Nginx, Apache and Docker on AWS](/guides/images/2_Node_Nginx_AWS_Example.png)

You will also be introduced to the [weavedns] service (https://github.com/weaveworks/weave/tree/master/weavedns#readme), which provides a simple way for containers to find each other using hostnames and requires no code changes.

This guide does not require any programming skills and will take about 15 minutes to complete.

## What You Will Use ##

* [Weave](http://weave.works)
* [Docker](http://docker.com)
* [Nginx](http://Nginx.org)
* [Ubuntu](http://ubuntu.com)
* [Amazon Web Services](http://aws.amazon.com)

##Before You Begin

This guide uses the [Amazon Web Services Command Line Interface (AWS CLI)](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) to manage and access AWS.

You need to have a valid [Amazon Web Services](http://aws.amazon.com) account, and the AWS CLI setup and configured before working through this guide. Amazon provides extensive documentation on how to setup the [AWS CLI](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-set-up.html).

* [Git](http://git-scm.com/downloads)
* [AWS CLI > 1.7.12 ](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html)

## Setting up the AWS Instances ##

To begin, clone the getting started repository and then change to the `aws-nginx-ubuntu-simple` directory:

~~~bash
    git clone https://github.com/weaveworks/guides
    cd guides/aws-nginx-ubuntu-simple
~~~

You will use the AWS CLI to set up and configure two AWS EC2 instances. For the purposes of this guide, the smallest available instances, t1.micro are used. 

A script to set up your initial environment with Weave and Docker is installed on Ubuntu.

If you would like to manually work through these steps, and for further details on the script, please refer to the _**Manual install on AWS section**_ at the end of this guide.

If you get errors regarding the AMI, set your preferred Ubuntu AMI value in the environment variable `AWS_AMI`.

    ./demo-aws-setup.sh

This script generates quite a lot of output, but once it is completed you will have two aws instances, running Ubuntu,
with Weave with Docker installed. You need the IP addresses of these instances to complete the rest of this guide. These are stored in an environment file `weavedemo.env` which we create during the execution of the `demo-aws-setup.sh`.

You will see something similar to the output below when you look at the weavedemo.env file. Please note these are *not* the
IP addresses for your demo, AWS dynamically allocates IP addresses to your instances.

    cat weavedemo.env
    export WEAVE_AWS_DEMO_HOST1=52.16.63.10
    export WEAVE_AWS_DEMO_HOST2=52.16.63.7
    export WEAVE_AWS_DEMO_HOSTCOUNT=2
    export WEAVE_AWS_DEMO_HOSTS=(52.16.63.10 52.16.63.7)

If you are using a bash or similar you can just source this file to use for the rest of this guide, otherwise please
subsititue the relevant IP address for $WEAVE_AWS_DEMO_HOST1 or 2.

    . ./weavedemo.env

## Introducing WeaveDNS ##

[WeaveDNS](https://github.com/weaveworks/weave/tree/master/weavedns#readme) answers name queries in a Weave network. WeaveDNS provides a simple way for containers to find each other: just give them hostnames and tell other containers to connect to those names. Unlike Docker 'links', this requires no code changes and works across hosts.

In this example you will give each container a hostname and use WeaveDNS to allow Nginx to find the correct container for a request.

## Nginx and a simple PHP application running in Apache ##

Nginx is a popular free, open-source, high-performance HTTP server and reverse proxy. It is frequently used as a load balancer. In this
example we will use Nginx to load balance requests to a set of containers running Apache.

While our example application is very simple, as a php application running on apache2 it resembles many real world applications
in use. We have created an Nginx configuration which will round-robin across the websevers our php application is running on.
The details of the Nginx configuration are out of scope for this article, but you can review it at [on github](https://github.com/weaveworks/guides/blob/master/nginx-ubuntu-simple/example/nginx.conf).

The two docker containers we use in this guide were created for an [earlier getting started guide](https://github.com/weaveworks/guides/blob/master/nginx-ubuntu-simple/README.md). If you would like further details on how these were created refer to the Dockerfile section at the end of this of guide.

## Application Portability with Weave and Docker ##

A key point to note is that we are using the application containers we created for our earlier example. By using WeaveDNS we do not have to make any changes to our containerized nginx configuration. In our earlier example we used three hosts running with Vagrant,
while in this example we will use two hosts on AWS.

Weave Net allow us to easily deploy our containers to a new infrastucture and configuration with no code changes, and without
the need to understand concepts such as ambassador containers and links.

## Starting the Example ##

To start the example run the script `launch-aws-nginx-demo.sh`. This will

* launch Weave Net on to each host
* launch six containers across our two hosts running an apache2 instance with our simple php site
* launch Nginx as a load balancer in front of the six containers

    ./launch-aws-nginx-demo.sh

If you would like to execute these steps manually the commands to launch Weave and WeaveDNS are

    ssh -i weavedemo-key.pem ubuntu@$WEAVE_AWS_DEMO_HOST1
    sudo weave launch
    sudo weave launch-dns 10.2.1.1/24
    
    ssh -i weavedemo-key.pem ubuntu@$WEAVE_AWS_DEMO_HOST2
    sudo weave launch $WEAVE_AWS_DEMO_HOST1
    sudo weave launch-dns 10.2.1.2/24

The commands to launch the application containers are

    ssh -i weavedemo-key.pem ubuntu@$WEAVE_AWS_DEMO_HOST1
    sudo weave run --with-dns 10.3.1.1/24 -h ws1.weave.local fintanr/weave-gs-nginx-apache
    sudo weave run --with-dns 10.3.1.2/24 -h ws2.weave.local fintanr/weave-gs-nginx-apache
    sudo weave run --with-dns 10.3.1.3/24 -h ws3.weave.local fintanr/weave-gs-nginx-apache
    
    ssh -i weavedemo-key.pem ubuntu@$WEAVE_AWS_DEMO_HOST2
    sudo weave run --with-dns 10.3.1.4/24 -h ws4.weave.local fintanr/weave-gs-nginx-apache
    sudo weave run --with-dns 10.3.1.5/24 -h ws5.weave.local fintanr/weave-gs-nginx-apache
    sudo weave run --with-dns 10.3.1.6/24 -h ws6.weave.local fintanr/weave-gs-nginx-apache

Note the `--with-dns` option and the `-h` option, `--with-dns` tells the container to use the `weavedns` service to resolve names and
`-h x.weave.local` allows the host to be resolvable with `weavedns`.

Finally launch the Nginx container

    ssh -i weavedemo-key.pem ubuntu@$WEAVE_AWS_DEMO_HOST1
    sudo weave run --with-dns 10.3.1.7/24 -ti -h nginx.weave.local -d -p 80:80 fintanr/weave-gs-nginx-simple

### What has happened? ###

At this point you have launched Weave Net on to all of your hosts, and connected six containers running a simple php application and Nginx together using Weave.

The Nginx container is publicly exposed as a http server on $WEAVE_AWS_DEMO_HOST1

## Testing the Example ##

To demonstrate our example we have provided a small curl script which will make http requests to the our Nginx container. We make
six requests so you can see Nginx moving through each of the webservers in turn.

    ./access-aws-hosts.sh

You should see output similar to the following:

    Connecting to Nginx in Weave on AWS demo
    {
        "message" : "Hello Weave - nginx example",
        "hostname" : "ws1.weave.local",
        "date" : "2015-02-25 14:35:53"
    }
    {
        "message" : "Hello Weave - nginx example",
        "hostname" : "ws2.weave.local",
        "date" : "2015-02-25 14:35:53"
    }
    {
        "message" : "Hello Weave - nginx example",
        "hostname" : "ws3.weave.local",
        "date" : "2015-02-25 14:35:53"
    }
    {
        "message" : "Hello Weave - nginx example",
        "hostname" : "ws4.weave.local",
        "date" : "2015-02-25 14:35:53"
    }
    {
        "message" : "Hello Weave - nginx example",
        "hostname" : "ws5.weave.local",
        "date" : "2015-02-25 14:35:53"
    }
    {
        "message" : "Hello Weave - nginx example",
        "hostname" : "ws6.weave.local",
        "date" : "2015-02-25 14:35:53"
    }

## Conclusions

You have now used Weave to deploy a containerised PHP application using Nginx across multiple hosts on AWS EC2.

## Manual Install on AWS ##

We make use of the [Amazon Web Services (AWS) CLI tool](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-welcome.html) to
manage and access AWS for this getting started guide. You will need to have a valid [Amazon Web Services](http://aws.amazon.com) account, and the AWS CLI setup and configured before working.

### Security Groups & Key Pairs ###

**1. Create a security group, for the example called `weavedemo`**

~~~bash
    aws ec2 create-security-group --group-name weavedemo --description "Weave Demo"
~~~

You then allow access to the host on ports 22 (for ssh), 80 (for http) and on 6783 (for weave). Please note the cidr option we pass in here.
For real world deployments you should limit this cidr to something more restricted, particuarly for ssh.

~~~bash
    aws ec2 authorize-security-group-ingress --group-name weavedemo --protocol tcp --port 22 --cidr 0.0.0.0/0
    aws ec2 authorize-security-group-ingress --group-name weavedemo --protocol tcp --port 80 --cidr 0.0.0.0/0
    aws ec2 authorize-security-group-ingress --group-name weavedemo --protocol tcp --port 6783 --cidr 0.0.0.0/0
    aws ec2 authorize-security-group-ingress --group-name weavedemo --protocol udp --port 6783 --cidr 0.0.0.0/0
~~~

Next create a key pair that allows access any EC2 instances which are associated with this security group.

~~~
    aws ec2 create-key-pair --key-name weavedemo-key --query 'KeyMaterial' --output text > weavedemo-key.pem
~~~

### Installing Weave and Docker ###

You will need to install docker on each host in turn. Please follow the offical instructions for installing Docker on Ubuntu on the [Docker website](https://docs.docker.com/installation/ubuntulinux/).


### Starting Your Instance ###

Now you start your instances. Please note the $AWS_AMI variable here, you need to give an image id for AWS to use.
In this demo we use Ubuntu, and a list of the available Trusty 14.04 images are available [here](http://cloud-images.ubuntu.com/trusty/current/).

    aws ec2 run-instances --image-id $AWS_AMI  --count 2 --instance-type t1.micro --key-name weavedemo-key --security-groups weavedemo

### Getting your IP Addresses ###

You will need to get the IP addresses of your instances in order to access them. You can get these details by looking at your
running instances.

    aws ec2 describe-instances --output text --filters "Name=instance-state-name,Values=running" "Name=instance.group-name,Values=weavedemo"

### Installing Weave and Docker ###

You will need to install docker on each host in turn, please follow the instructions on the [Docker website]().
To install weave you will need to ssh into each host and run the following commands.

    sudo curl -L git.io/weave -o /usr/local/bin/weave
    sudo chmod a+x /usr/local/bin/weave

## The Dockerfiles ##

We have included the two Dockerfiles we used for creating our containers.  A full discussion of Dockerfiles is out of
scope for this guide, for more information refer to the offical guide on [docker.com](https://docs.docker.com/reference/builder/).

Our containers have been built from the offical [Nginx](https://registry.hub.docker.com/_/nginx/) and [Ubuntu](https://registry.hub.docker.com/_/ubuntu/) images, and pushed to the Docker Hub

If you want to experiment with these Dockerfiles it is easiest to git clone the [Weave Getting Started repository](http://github.com/weaveworks/guides) on the host. You may need to install git.

### Nginx ###

    ssh -i weavedemo-key.pem ubuntu@$WEAVE_AWS_DEMO_HOST1
    git clone http://github.com/weaveworks/guides
    cd guides/aws-nginx-ubuntu-demo
    sudo docker build -f Dockerfile-simple-nginx .

When you run docker build you will see out similar to

    Sending build context to Docker daemon 34.82 kB
    Sending build context to Docker daemon
    Step 0 : FROM nginx
     ---> 4b5657a3d162
    Step 1 : RUN rm /etc/nginx/conf.d/default.conf
     ---> Using cache
     ---> 27365a3d665f
    Step 2 : RUN rm /etc/nginx/conf.d/example_ssl.conf
     ---> Using cache
     ---> 4a98cc012cfd
    Step 3 : COPY nginx.conf /etc/nginx/conf.d/default.conf
     ---> Using cache
     ---> 5583ba8e1c8a
    Successfully built 5583ba8e1c8a

To run this container take the id from the ~Successfully built~ output and use Weave or Docker to launch the container.

    sudo weave run 10.3.1.64/24 5583ba8e1c8a

or to just launch the container

    sudo docker run 5583ba8e1c8a

### Apache and PHP Application ###

    ssh -i weavedemo-key.pem ubuntu@$WEAVE_AWS_DEMO_HOST1
    git clone http://github.com/weaveworks/guides
    cd guides/aws-nginx-ubuntu-demo
    sudo docker build -f Dockerfile-simple-apache .
