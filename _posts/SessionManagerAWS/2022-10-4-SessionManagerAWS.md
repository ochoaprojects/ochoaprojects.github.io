---
title: Session Manager for AWS
date: 2022-10-04 12:00:00 -500
categories: [Cloud Providers,Amazon Web Services (AWS)]
tags: [aws,amazon,session manager,ssh tunnel,iam,ec2]
---

Session Manager is the useful AWS tool that you might not be thinking about. If you've ever run something like ProxMox you understand just how handy it is to be able to click "console" for any of your VMs and instantly be transported to that VM as if you were sitting directly infront of the real deal with ease. This is "almost" that, but with a little more power in my opinon.

## Table of Contents
- [Create IAM Role](#create-iam-role)
- [Launch EC2 Instance](#launch-ec2-instance)
- [Configure CloudWatch](#configure-cloudwatch)
- [Install AWS CLI and AWS Session Manager Plugin](#install-aws-cli-and-aws-session-manager-plugin)
- [Create IAM Policy and IAM User and IAM Group](#create-iam-policy-and-iam-user-and-iam-group)
  - [IAM Policy](#iam-policy)
  - [IAM Group](#iam-group)
  - [IAM User](#iam-user)
- [SSH to EC2 Instance](#ssh-to-ec2-instance)
- [Port Forward from EC2 to localhost](#port-forward-from-ec2-to-localhost)
  - [Setting Up EC2 Instance](#setting-up-ec2-instance)
  - [Starting Port Forward](#starting-port-forward)

> The Session Manager is part of the AWS Systems Manager and if you want to know more or read up on the offical documentation, that can be found [here.](https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html).
{: .prompt-info }

# Create IAM Role
When you create an AWS account, you begin with one sign-in identity that has complete access to all AWS services and resources in the account. This identity is called the AWS account root user and is accessed by signing in with the email address and password that you used to create the account. It is strongly recommended that you do not use the root user for your everyday tasks. Safeguard your root user credentials and use them to perform the tasks that only the root user can perform.

In this procedure, you use the AWS account root user to create your first user in AWS Identity and Access Management (IAM). You add this IAM user to an Administrators group, to ensure that you have access to all services and their resources in your account. The next time that you access your AWS account, you should sign in with the credentials for this IAM user. As a best practice, create only the credentials that the user needs. For example, for a user who requires access only through the AWS Management Console, do not create access keys.

Click Create Role.
![Click Create Role](/project-assets/SessionManagerAWS/click-create-role.png)

Select AWS Service and choose EC2 at the bottom.
![Select Trusted Entity](/project-assets/SessionManagerAWS/select-trusted-entity.png)

Search for "AmazonSSMFullAccess", tick the box, click next. 
![Add Permissions](/project-assets/SessionManagerAWS/add-permissions.png)

Give your role a name, review, and create it.
![Review and Create](/project-assets/SessionManagerAWS/review-and-create.png)

# Launch EC2 Instance
AWS Systems Manager Agent (SSM Agent) is Amazon software that can be installed and configured on an EC2 instance, an on-premises server, or a virtual machine (VM). SSM Agent makes it possible for Systems Manager to update, manage, and configure these resources.

If the Amazon Machine Image (AMI) type you choose in the first procedure doesn't come with SSM Agent preinstalled, manually install the agent on the new instance before it can be used with Systems Manager. If SSM Agent isn't installed on the existing EC2 instance you choose in the second procedure, manually install the agent on the instance before it can be used with Systems Manager.

In most cases, SSM Agent is preinstalled on AMIs provided by AWS for the following operating systems (OSs):
- Amazon Linux Base AMIs dated 2017.09 and later
- Amazon Linux 2
- Amazon Linux 2 ECS-Optimized Base AMIs
- Amazon EKS-Optimized Amazon Linux AMIs
- macOS 10.14.x (Mojave), 10.15.x (Catalina), and 11.x (Big Sur)
- SUSE Linux Enterprise Server (SLES) 12 and 15
- Ubuntu Server 16.04, 18.04, and 20.04
- Windows Server 2008-2012 R2 AMIs published in November 2016 or later
- Windows Server 2016, 2019, and 2022

Navigate to `EC2 > Instances` and click "Launch Instances" in the top right hand corner and name your EC2 whatever you want.
![Create EC2](/project-assets/SessionManagerAWS/create-ec2.png)

The defaults here should be good and keep you in a "Free Tier" for learning purposes. However, the `most important` part here is the `Advanced details` at the bottom of this page. Expanding this will reveal the `IAM instance profile` option. Click the drop down and choose the IAM Role we made earlier.

![IAM Instance Profile](/project-assets/SessionManagerAWS/iam-instance-profile.png)

Now we simply launch our instance.
![Launch Instance](/project-assets/SessionManagerAWS/launch-instance.png)

Since we are using the SessionManager to connect to this EC2 instance for this exmaple we are able to complete to proceed without using a key pair.
![No Key Pair](/project-assets/SessionManagerAWS/no-key-pair.png)

With what we have so far, it should be possible for us to head over to `AWS Systems Manager > Session Manager` within AWS console and click `Start Session` with the EC2 instance we just made.
![Start Session](/project-assets/SessionManagerAWS/start-session.png)
![Target Instances](/project-assets/SessionManagerAWS/target-instances.png)
![Connected Via SSM](/project-assets/SessionManagerAWS/connected-via-ssm.png)

# Configure CloudWatch
> ***What is CloudWatch?--***  Amazon CloudWatch is a monitoring and observability service built for DevOps engineers, developers, site reliability engineers (SREs), IT managers, and product owners. CloudWatch provides you with data and actionable insights to monitor your applications, respond to system-wide performance changes, and optimize resource utilization. CloudWatch collects monitoring and operational data in the form of logs, metrics, and events. You get a unified view of operational health and gain complete visibility of your AWS resources, applications, and services running on AWS and on-premises. You can use CloudWatch to detect anomalous behavior in your environments, set alarms, visualize logs and metrics side by side, take automated actions, troubleshoot issues, and discover insights to keep your applications running smoothly.
{: .prompt-info }

This is an `optional step` but it is also very powerful. This will allow you to output your sessions as logs into CloudWatch so that you can review them later.

Start by going to `Cloud Watch > Log Groups` in the console and `Create log group`.
![Create Log Group](/project-assets/SessionManagerAWS/create-log-group.png)

Give it whatever name you prefer, in this example I am going with "ssm-session". Click "Create"
![Log Group Name](/project-assets/SessionManagerAWS/log-group-name.png)

Now we need session manager to use the log we just created. Navigate to `AWS Systems Manager > Session Manager` and click on `Preferences` then `Edit`.
![Session Manager Edit Preferences](/project-assets/SessionManagerAWS/session-manager-edit-preferences.png)

> I disable "Enforce Encryption" for the sake of simplicity in this example.
{: .prompt-info }

First we are going to `Enable` CloudWatch Logging. Next we are going to keep the **Recommended** default of `Stream Session Logs`. Then we are going to *uncheck* the `Enforce Encryption` for this log group. Lastly we are going to choose the log group we just made in our CloudWatch in the last step.
![CloudWatch Logging Settings](/project-assets/SessionManagerAWS/cloudwatch-logging-settings.png)

# Install AWS CLI and AWS Session Manager Plugin
For this example I am on Mac. However, this can be achieved with both Windows and Linux. My recommendation for windows is to use [Chocolately](https://chocolatey.org/install). When using Linux you can of course use your native package manager such as RPM or APT.

While using Mac I try to leverage [Homebrew](https://brew.sh/) when I can.

Here are the two commands that will get AWS CLI and AWS Session Manager Plugin installed on your machine for Mac.

```bash
brew install awscli
```

```bash
brew install --cask session-manager-plugin
```

# Create IAM Policy and IAM User and IAM Group

## IAM Policy
To create an IAM Policy we navigate to `IAM
Policies` and click "Create Policy".

![Create IAM Policy](/project-assets/SessionManagerAWS/create-iam-policy.png)

Choose the `JSON` tab and paste in the code below.
![Create IAM Policy With JSON](/project-assets/SessionManagerAWS/create-iam-policy-with-json.png)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:StartSession"
            ],
            "Resource": [
                "arn:aws:ec2:<region>:<account-id>:instance/*"
            ],
            "Condition": {
                "StringLike": {
                    "ssm:resourceTag/service": [
                        "proxy"
                    ]
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:StartSession"
            ],
            "Resource": [
                "arn:aws:ssm:<region>::document/AWS-StartPortForwardingSession"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:TerminateSession"
            ],
            "Resource": [
                "arn:aws:ssm:*:*:session/${aws:username}-*"
            ]
        }
    ]
}
```
> Note the "AWS-StartPortForwardingSession" part in the above JSON. Session Manager allows for SSH tunneling and it is enabled here. This feature can be very useful as demontrated in this picture.
![Session Manager SSH Tunnel](/project-assets/SessionManagerAWS/session-manager-ssh-tunnel.png)
This command tells SSH to connect to `instance` as user `ec2-user`, open port 9999 on my local laptop, and forward everything from there to `localhost:80` on the instance. When the tunnel is established, I can point my browser at `http://localhost:9999` to connect to my private web server on port 80.
{: .prompt-info }

Be sure to update `<region>` and `<account-id>` to the values required for your environment.

Review and name your IAM Policy and click "Create Policy". For this example I am giving the name "DemoSessionPolicy".
![Review and Name IAM Policy](/project-assets/SessionManagerAWS/review-and-name-iam-policy.png)

## IAM Group
Next head over to `IAM > User groups` and click "Create Group".
![Create User Group](/project-assets/SessionManagerAWS/create-user-group.png)

Name your IAM Group. In this example I am naming it "SSMUsers" because whatever user I add to this group will have access to the Session Manager and be able to use the features we add earlier such as SSH tunneling.
![Name IAM Group](/project-assets/SessionManagerAWS/name-iam-group.png)

Attach the IAM Policy that we made earlier to the IAM Group. Customer managed polcies, such as the one we made, will be listed at the top of the list for convenience. Click "Create Group" at the bottom when complete.
![Attach IAM Policy](/project-assets/SessionManagerAWS/attach-iam-policy.png)

## IAM User
Next head over to `IAM > Users` and click "Add User".
![IAM Users Add User](/project-assets/SessionManagerAWS/iam-users-add-user.png)

Name your user and choose `Access Key - Programmatic access` for AWS credential type. We choose this because this user does not need to access the console and only want to be access the AWS resources using SSM through things such as the AWS CLI. Click "Next: Permissions" at the bottom when complete.
![Configure IAM User](/project-assets/SessionManagerAWS/configure-iam-user.png)

Since we already have a group set up from the previous step, we can simply add our new user to the group we just created in this step by ensuring `Add user to group` is selected, checking the box next to the group we want to be added to, and then clicking "Next: Tags" at the bottom.
![Add IAM User to Group](/project-assets/SessionManagerAWS/add-iam-user-to-group.png)

I don't add any tags for this exmaple. Simply click "Next: Review".

Click "Create User".
![Click Create User](/project-assets/SessionManagerAWS/click-create-user.png)

> IMPORTANT! AWS Console will immediately provide us the `Access key ID` and `Secret access key` on the next screen because we chose the "Programmatic" access for this user. It is very important that you document this information somewhere safe that you know you will be able to access and find again. There is a `Download .csv` option here which is a useful quick and easy step. However, I recommend using something like the `AWS Secrets Manager` for storing information such as this (This feature is not free and will cost a small fee). `If you close this screen without saving the information, you will *NOT* be able to retrieve this information again and you *WILL* need to recreate a new user to recreate the information.`
{: .prompt-danger }

Document your `Access key ID` and `Secret access key` someplace safe and where you can find it again.
![Save Access and Secret Key IDs](/project-assets/SessionManagerAWS/save-access-and-secret-key-ids.png)

# SSH to EC2 Instance
Now that we have an IAM User, Role, Group, Policy set up and AWS CLI plus Session Manager Plugin installed we can ssh into our EC2 resource with the AWS CLI.

First we need to configure our AWS CLI witht the following command.
```bash
aws configure
```

Next we will provide that `AWS Access Key ID` and `AWS Secret Access Key` that we definitely documented earlier.

Afterwards it will ask for the following-
`Default region name`: us-east-2 (This is my setting)
`Default output format`: json (This is default and what I used)

To start a session run the following command. Replace `<instance-id>` with your own instance-id.
```bash
aws ssm start-session --target <instance-id>
```

![Successful SSH Connection](/project-assets/SessionManagerAWS/successful-ssh-connection.png)

# Port Forward from EC2 to localhost

## Setting Up EC2 Instance
To demonstrate the port forwarding feature we will use a simple NGINX server on our demo ec2 instance.

Let's start by installing NGINX with the linux package manager by using the following command.
```bash
sudo amazon-linux-extras install nginx1 -y
```

I chose the Linux AMI provided by Amazon and therefore I am not using something such as yum or apt. Instead it uses "amazon-linux-extras".

After NGINX is installed, let's us systemd to start and check our server's status.

```bash
sudo systemctl start nginx
```

```bash
sudo systemctl status nginx
```

Now we will see that our NGINX server is active and running.

## Starting Port Forward
Let's start a port forward session with the following command. Be sure to enter the correct `ec2-instance-id` for your command.

```bash
aws ssm start-session \
    --target <ec2-instace-id> \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["80"], "localPortNumber":["8080"]}'
```

Your terminal will display that it is `Waiting for connections...`

Let's make that connection! Simply open your browser from your local machine that you are running the AWS SSM command from and navigate to `http://localhost:8080` and you will be greeted with "Welcome to NGINX on Amazon Linux!"

We have successfully created and used our SSH Tunnel with the SSM!