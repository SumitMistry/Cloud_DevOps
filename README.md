
# Cloud Deployment Architecture & Micro-service deployment 
## Introduction
![enter image description here](https://raw.githubusercontent.com/sumitJRVS/Cloud_DevOps/master/cloudArchi.png)

This project acts as an example of how to deploy a micro-service to the cloud.  This document will explain the various tools and techniques used to streamline the deployment process resulting in an efficient, effective, and (mostly) secure methodology which can be used for many projects.  Two different techniques will be discussed, but both approaches will provide the same functionality in regards to the end user. 

Included in this repository is a previously developed project of a SpringBoot managed micro-service, which can be found in full [here](https://github.com/andrew-laniak-fuoco/trading_api), along with a description of it's functionality and architecture.  It is recommended for anyone with little to no deployment experience to use the included application in order to learn how these deployment processes work.

The trading application specifically will be built with three working pieces giving it a three tiered architecture.  The three parts are client, application server, and database.  The two deployment techniques will make use of a PostgreSQL database for the database tier, as the trading application expects the mentioned database.

-----------

## Docker Deployment

![enter image description here](https://raw.githubusercontent.com/sumitJRVS/Cloud_DevOps/master/dockerGeneral.png)
As stated earlier, there will be two techniques discussed as to how to deploy our application.  The first technique is accomplished via Docker.  Note that the following is a fairly lengthy and manual process, but it gives good insight into how Docker works and how it can be used for other similar projects.  The architecture for this deployment scheme is simple yet effective.
Before starting, it is required to have Maven, Docker and PostgreSQL installed and ready to use.  These are all very common pieces of software so finding some guide online for your specific package manage should be easy.

## Docker
![enter image description here](https://raw.githubusercontent.com/sumitJRVS/Cloud_DevOps/master/components.png)
The first step is to start Docker on your machine:
`systemctl start docker`
It is also good to run the following command which will start docker automatically when the host machine boots up:
`systemctl enable docker`

Next is to create a network which will be used to connect the application server tier with the database tier.  Simply run the command and docker will create a new bridge network which can be leverage to have different Docker containers communicate with each other:
`docker create network --driver bridge [network-name]`

### PostgreSQL Container
Before creating a Docker container for the application, we first need to set up the database.  Run the following command to create a Docker container which will hold a PostgreSQL database:
`docker run --name psql -e POSTGRES_PASSWORD=[password] -e POSTGRES_DB=[database-name] -e POSTGRES_USER=[username] --network [network-name] -d -p 5432:5432 psql`

The start of this command (`docker run`) creates the image.  All the `-e` flags are setting environmental variables for the database.  These variables will also be used by the application to make a connection to the database, so be sure to record them somewhere.  `--network [network-name]` tells Docker to link this container to the specified network, and `-p 5432:5432` declares the port the container will listen on.  

Finally the database needs to be set up with it's schema.  The sql folder holds the necessary sql commands to initialize the database for use by the trading application.  These files can be run with the following commands:
`psql -h localhost -U [username] -f sql/init.sql`
`psql -h localhost -U [username] -d jrvstrading -f sql/schema.sql`

### Application Container
![enter image description here](https://raw.githubusercontent.com/sumitJRVS/Cloud_DevOps/master/trading-aws.png)
Assuming we will use the trading API project, start by downloading the source.  Run a git command to pull the source from GitHub.
`git clone https://github.com/andrew-laniak-fuoco/Cloud_Deployment`
cd into the new directory then package the project into a jar using the included Dockerfile with the following command.
`docker build -t trading-api ./`

Then we need to create the docker image.  Running the following command will create an image, for the trading application
`docker run --name trading -e PGUSER=${PG_NAME} -e PGURL=${PGURL} -e PGPASSWORD=${PGPASSWORD}-e iexToken=${iexToken} --network [network-name] -p 5000:8080 -t trading-api`

This command works much like the PostgreSQL command with a few differences.  `--restart unless-stopped` is a flag that will start the application when the machine starts up, as well it will reboot in the case of a crash.  `-p 5000:8080` declares the port the container will listen on, just like before but this acts as a port translation.  As a default SpringBoot applications will listen on port 8080 for requests, but the docker container will listen on port 5000.  This docker option will open the container to port 5000, then internally it will convert to port 8080 for the running application.   

## Cloud Architecture
![enter image description here](https://raw.githubusercontent.com/sumitJRVS/Cloud_DevOps/master/docker1.png)

To further develop our deployment scheme we'll make use of the cloud.  The weak point in this architecture is it's single entry point.  As this architecture was designed as a foundation to work with, the port to the load balancer is publicly open.  However, outside of this vulnerability the rest of the system is safe guarded by the security groups in place, ensuring incoming traffic to any component can only come from another component within the system.

Before going over each piece of the design, it is important to understand what a security group is (security groups are denoted as red dashed boxed).  Security groups define what kind of traffic can be received or sent from any object within the AWS framework.  Here we have unique security groups for the load balancer, EC2 instances, and the database.  For this project out going traffic was not limited, but incoming traffic was heavily controlled.  Details towards each individual security group will be discussed below.

The first component of this design is the load balancer.  For simplicity, the connection port for this component was 80, which is the default port used by HTTP allowing quick access through web browsers.  The load balancer will take any request given to it and forward it to one of the EC2 instances.  The load balancer will send it's requests on port 5000 which is expected both by the EC2 security group and the Docker containers.

The final component is a database.  Specifically the included project uses a PostgreSQL server which looks for requests on port 5432.  The database is access through JDBC by the application.  The database's security group expects traffic specifically from the security group used by the auto-scaling group.  This ensures that the data cannot be access by anything except the application being run which allows the developer to control what kind of information can be seen by any user.  

In order to use this cloud architecture using Docker, only the application container is needed as the cloud service will supply the database.  It is important to set up the application to run automatically whenever an EC2 instance starts.  Adjust the command to create the application image like so:
`docker run --restart unless-stopped --name trading -e PGUSER=${PG_NAME} -e PGURL=${PGURL} -e PGPASSWORD=${PGPASSWORD}-e iexToken=${iexToken} --network [network-name] -p 5000:8080 -t trading-api`

The new flag, `--restart unless-stopped`, will boot up the application when the server starts, and it will reboot in the case of a crash.  This flag is critical to making the cloud architecture work with an auto-scaling group as it would be very impractical to have to start the application manually whenever a new instance was started.

## Elastic Beanstalk Deployment
![enter image description here](https://raw.githubusercontent.com/sumitJRVS/Cloud_DevOps/master/jenkins.png)

Here is an architecture diagram of the second deployment technique.  On the surface it looks like a lot more work, however Elastic Beanstalk (EBS) handles most of this work.  Before going through the instructions on how to develop this design, let us discuss the difference in architecture between this set up and the Docker deployment technique.

Starting in the middle, this technique introduces a new tool that remove the need for Docker at all.  Instead of using a container for our application, EBS provides automatic deployment for a number of different file types.  In this case we only need to upload the application Jar file to EBS and it will handle the rest regarding the EC2 instances.  

EBS additionally handles managing security groups, the load balancer, the auto-scaling group, as well it has the ability to provision a new database (an existing database can be used too).  With the Docker cloud set up all these components needed to be provisioned manually, yet here EBS takes care of the basic work for us.

On the left side we have something new, which is used to streamline the process of updating whatever software application we are using.  For a Docker based deployment, one would need to make a new Docker container and set up the security group to use new EC2 instances that use the new container.  Overall not a long process, but it would be bothersome to have to do every time an update was required.  In this technique we have a new EC2 instance that, instead of running the application it runs NGINX which allows HTTP connection to a Jenkins instance.  

Jenkins automates the updating process by establishing a connection to both GitHub and EBS.  Jenkins will detect if any changes has been made to the project on GitHub, and if there has it will send a new Jar file directly to EBS.  EBS will then update all the EC2 instances with the new application file within minutes.

On the right there are two replications of the full cloud architecture.  This is because Jenkins has the ability to manage different branches as well when updating, and EBS has the ability to manage different environments within a single application.  The diagram is expressing this by showing one Jenkins and EBS instance can manage two application environments.  In this case it is representing how a developer can have one environment for testing while the other is for release versions of the application.  Theoretically one could have more environments in case that different development teams are using the application.  Since each environment has it's own database they can all be seen an independent from each other.

