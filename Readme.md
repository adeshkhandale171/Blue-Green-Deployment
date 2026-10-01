# Production-Ready Blue-Green Deployment using Jenkins and Application Load Balancer

In this Project, we will build a Production-Ready Blue-Green Deployment environment using AWS EC2, Application Load Balancer, Jenkins, Nginx, and GitHub.

## 1. Project Overview

This project demonstrates how to implement a Blue-Green Deployment strategy using two separate application environments.

The **Blue Environment** represents the currently running production version, while the **Green Environment** is used to deploy and test the new application version.

Jenkins automatically pulls the application code from GitHub and deploys the new version to the Green server. Before production traffic is switched, Jenkins performs a health check on the Green environment.

If the health check is successful, Jenkins automatically switches the Application Load Balancer traffic from Blue to Green.

If the health check fails, the deployment is marked as failed and the ALB traffic is automatically rolled back to the Blue environment.

## Main Objectives

- Create separate Blue and Green environments.
- Deploy different application versions on Blue and Green EC2 instances.
- Configure an Application Load Balancer.
- Create separate Blue and Green Target Groups.
- Use Jenkins for automated deployment.
- Pull application code from GitHub.
- Perform a health check before switching production traffic.
- Switch ALB traffic from Blue to Green after successful validation.
- Automatically roll back traffic to Blue when the Green deployment fails.

## 2. Architecture

**Architecture Diagram**

![src1](./img/Blue-Green%20Deployment%20Architecture.png)

**Architecture Explanation**

The project uses two separate EC2 environments called Blue and Green. The Blue environment contains the currently active application version, while the Green environment is used for deploying and validating the new application version. Each EC2 instance runs Nginx and is registered with its own target group.

The Application Load Balancer acts as the entry point for users and forwards traffic to either the Blue or Green target group. Jenkins connects to the Green EC2 server, deploys the new application version, and performs a health check. If the health check passes, Jenkins changes the ALB listener to forward traffic to Green-TG. If the health check fails, Jenkins skips the traffic-switching stage and rolls the ALB listener back to Blue-TG.

## 3. Requirements
* Amazon EC2       
* Application Load Balancer
* Target Groups
* Jenkins
* GitHub
* Nginx Web Server


## 4. Steps
### 1. Infrastructure Setup
#### 1. Create 2 EC2 Instances

Create two EC2 instances for the Blue and Green environments.

* __Blue Environment__
    * Runs the current/Version 1 application.
    * Initially receives production traffic from the ALB.
* __Green Environment__
    * Used for deploying the next version of the application.
    * It will not be manually configured with the new application version.
    * Jenkins will automatically deploy the new version to this server.

#### 2. Install Web Server

Install Nginx on the Blue server.
```
sudo dnf update -y
sudo dnf install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
sudo systemctl status nginx
```

#### 3. Deploy Version 1 Application

* Copy Version 1 of the application to the Blue server.

* The Blue server now contains Version 1 of the application.


__Note: We do not manually deploy Version 2 to the Green server. The next version will be deployed automatically by the Jenkins pipeline.__

### 2. Load Balancer Configuration
#### 1. Create Target Groups

Create two target groups for the two environments.

* __Blue Target Group__

_Create:_

   * Target Group Name: Blue-TG
   * Target Type: Instances
   * Protocol: HTTP
   * Port: 80
* Health Check Protocol: HTTP
* Health Check Path: /

__Register the Blue EC2 instance with Blue-TG.__

* __Green Target Group__

_Create:_
* Create Same as the Blue Target Group.

__Register the Green EC2 instance with Green-TG.__

The target groups allow the ALB to manage Blue and Green environments separately.

### 2. Create Application Load Balancer

Create an Application Load Balancer.

_Configure:_

* Load Balancer Type: `Application Load Balancer`
* Scheme: `Internet-facing`
* IP Address Type: IPv4
* Select the required VPC and subnets.
* Configure the security group to allow HTTP traffic on port 80.
### 3. Configure Listener

Create an HTTP listener:

* Protocol: HTTP
* Port: 80

Initially configure the listener to forward traffic to:

    `Blue-TG`

### 4. Attach Appropriate Instances

Verify that:

Blue EC2 → Blue-TG

Green EC2 → Green-TG

![src3](./img/Screenshot%202026-10-01%20202618.png)
### 5. Test Version 1

Copy the ALB DNS name and open it in a web browser.


`http://http://blue-green-alb-1643933889.us-west-1.elb.amazonaws.com/`


![src2](./img/Screenshot%202026-10-01%20160738.png)
The browser should display Version 1 from the Blue environment.

## __Note: At this stage, Version 1 is serving production traffic. The next version of the website/project will now be deployed using the Blue-Green Deployment process.__

### 3. Upload the Next Version to GitHub

Prepare the next version of the application on the local machine.
```
GitHub REPO/
├── index.html
├── health.html
└── Jenkinsfile
```

* The index.html contains Version 2.
* Push the updated project to GitHub.
* GitHub will now contain the new version that Jenkins will pull during deployment.

### 4. Jenkins Server Setup
#### 1. Create Jenkins EC2 Instance

Create another EC2 instance for Jenkins.

This server will be responsible for:

* Pulling code from GitHub
* Connecting to the Green server
* Deploying the application
* Performing health checks
* Switching ALB traffic
* Performing rollback when required

#### 2. Install Jenkins

Install Java and Jenkins on the Jenkins EC2 server.
Also install the `aws cli`


### 3. Access Jenkins

Open Jenkins in the browser using:

`http://<JENKINS-PUBLIC-IP>:8080`

Complete the initial Jenkins setup.

### 4. Install Required Plugins

Install the plugins required for this project, such as:

* Git
* SSH Agent
* Pipeline
* Pipeline: Stage View
* GitHub-related plugins, etc.

### 5 Configure SSH Communication with Green Server

Jenkins needs SSH access to the Green EC2 instance so that it can automatically deploy the application.

__Create Jenkins Credentials__

In Jenkins:
```
Manage Jenkins
    ↓
Credentials
    ↓
Global
    ↓
Add Credentials
```
Select:

`Kind: SSH Username with private key`

Configure:
```
Username: ec2-user
ID: ec2-user
```
Add the private SSH key used to connect to the Green EC2 instance.

The private key is the .pem key downloaded when the EC2 key pair was created.




The Jenkinsfile can then use this credential:
```
sshagent(['ec2-user']) {
    // SSH deployment commands
}
```

### 6. Give Jenkins Permission to Operate the ALB

Jenkins also needs AWS permissions to switch traffic between the Blue and Green target groups.

Attach an IAM Role to the Jenkins EC2 instance.

The flow becomes:
```
Jenkins
   ↓
AWS CLI
   ↓
ALB Listener
   ↓
Blue-TG / Green-TG
```

## 5. Jenkins Pipeline

Create a Jenkinsfile in the GitHub repository.

Configure Jenkins to use:
```
Pipeline
    ↓
Pipeline script from SCM
    ↓
Git
    ↓
GitHub Repository
    ↓
Branch: main
    ↓
Script Path: Jenkinsfile
```
The pipeline performs the Blue-Green deployment process.

![src5](./img/Screenshot%202026-10-01%20202548.png)

## 6. Verify Version 2 Using ALB DNS

After the Jenkins pipeline completes successfully, open the same ALB DNS name in the browser:
`http://http://blue-green-alb-1643933889.us-west-1.elb.amazonaws.com/`


![src4](./img/Screenshot%202026-10-01%20201604.png)
## 7. Rollback Logic

The rollback functionality is tested manually by intentionally making a temporary change in the Jenkinsfile so that the Green health check fails.

`sudo rm -f /usr/share/nginx/html/health.html`

__If Health Check Fails__
  * The traffic must remain/revert to `Blue-TG` 

The rollback flow is:
```
Jenkins
   ↓
Deploy Version 2 to Green
   ↓
Health Check
   ↓
FAILED
   ↓
Rollback
   ↓
Blue-TG
   ↓
Blue EC2
   ↓
Version 1
```
After rollback, verify the ALB listener and open the ALB DNS again.

The website should display Version 1, confirming that production traffic has returned to the previous environment.

## Important: The health-check failure change is only for testing the rollback mechanism. It should be removed from the final Jenkinsfile.

## 5. Lessons Learned
1. Understanding Blue-Green Deployment architecture.
2. Configuring Application Load Balancer and target groups.
3. Deploying applications using Jenkins and SSH.
4. Performing application health checks.
5. Automating ALB traffic switching using AWS CLI.
6. Implementing and testing rollback logic.
7. Using IAM Roles for AWS access from an EC2-based Jenkins server.

## 6. Project Summary

This project demonstrates a Blue-Green Deployment strategy using AWS EC2, Application Load Balancer, Jenkins, and GitHub. Version 1 initially runs on the Blue environment while the new Version 2 is deployed automatically to the Green environment through Jenkins. Jenkins performs a health check before switching ALB traffic to Green. If the deployment is successful, users receive Version 2. If the health check fails, Jenkins marks the deployment as failed and rolls traffic back to the Blue environment. This provides a controlled deployment and rollback process without manually changing the production application.