# 3 Tier Architecture using AWS

Welcome to my breakdown of 3-tier architecture using AWS. In this document, I will provide a detailed account of the process I followed while designing this architectural pattern. Before I dive into this project, I want to thank you for checking out my page. I encourage you to check out some of my other projects like the [Cloud Resume Challenge](https://github.com/oleskatony/cloudresumechallenge) and the [AWS Cloud Architecture Project](https://github.com/oleskatony/vpc-architecture), additionally you check check out my [website](https://antoleska.net/) if you want to see more!

⚠️ **WARNING: Please be aware that some services used in this project fall outside of the AWS Free-Tier. If you plan on recreating this project, please be cautious about the services you provision in AWS. Make sure to remove all infrastructure properly upon completion to avoid unnecessary charges to your account.** ⚠️

The 3-tier architecture approach is commonly used in software development, particularly in web applications. It separates the different components of an application into three distinct layers (or tiers), each responsible for specific functionality. These tiers are:

![3-tier-arch drawio](https://github.com/oleskatony/3-tier-architecture/assets/128739036/72966b99-d05f-48c0-97ff-685f3c430eb0)



## The Web Tier (Presentation Tier)

The web tier is responsible for handling user interaction and displaying information to the user. It typically includes web pages, mobile apps, or any other interface through which users interact with the application. The web tier communicates with the other layers to retrieve data and process user inputs.

## The Application Tier

The application tier contains the business logic or rules that define how the application operates. It processes requests from the web tier, performs data validation, executes complex calculations, and orchestrates the flow of data between different components. It acts as the intermediary between the web tier and the database tier.

## The Database Tier

The database tier is responsible for managing the storage and retrieval of data. It typically includes databases or other data storage systems. The database tier receives requests from the application tier, performs database operations such as reading or writing data, and returns the results back to the application tier.
# Process Overview

Let's quickly take a look at the steps needed to create a structure like the diagram above:

1. Create and configure the VPC with the correct amount of subnets allocated in the correct AZs and deploy NAT Gateway.
2. Create the Auto scaling group, launch template, and application load balancer, and security group for the web tier.
3. Provision the same resources as seen in Step 2, but configure them for the application tier.
4. Create resources for the database tier, including the DB subnet group, Multi-AZ RDS (MySQL), and security group.
5. Configure route tables by consolidating and associating the route tables with their correct subnets.

# Step 1: VPC

By using the "VPC and more" option when creating our VPC, we can save a lot of time spent manually configuring and deploying resources such as the NAT Gateway, Internet Gateway, and many subnets needed for this project. We use a `10.0.0.0/16` CIDR block for our VPC to have enough space for allocating subnets for our multi-tier infrastructure. Be sure to also enable "DNS hostnames" and "DNS resolutions".

Here you can see the configuration of the subnets. We use `10.0.0.0/24 - 10.0.5.0/24` (6 subnets in total). We can break the subnets down by layers to make it easier to understand. There are 2 subnets for each layer, divided between 2 Availability Zones. Out of the 6 subnets, 2 are public, and the remaining 4 are private.

![](https://i.imgur.com/FKx5S8U.png)

You might notice an abundance of routing tables. This is the major drawback of using the VPC and more option, as you will have to later configure the routing tables if you don't want unused or redundant routing tables in your VPC. In step 5, we will go back and clean all the routing tables up. Using other methods to provision AWS infrastructure could be a great way to prevent this issue (AWS CLI, or IaC tools like CloudFormation & Terraform).

Finally , you need to enable "auto-assign public IPv4 addresses" on the two public subnets. You find this option in the actions when the subnet is selected after clicking "edit subnet settings".
![](https://i.imgur.com/lROXihn.png)
# Step 2: Web Tier

To get started building the internet-facing web tier, you can start by creating a security group to use for the EC2 instances that you will provision. Enable SSH, HTTP, and HTTPS. This security group is over permissive, but for the sake of this lab, that shouldn't be a big problem. Name it something like `web-sg`.

![](https://i.imgur.com/0NFGiKP.png)

On the EC2 dashboard, create an Auto Scaling Group (ASG). There are a few prerequisites that are needed to be provisioned as you configure your ASG. You will need to create a launch template and change a few of the launch configurations for the template as follows:

- AMI: Amazon Linux 2023
- Instance Type: t2.micro
- Key Pair: Create New
- Subnet: Don't Include in Launch Template
- Security Group: web-sg
- VPC: The VPC from Step 1
- User Data: Paste the following code in the user data section (found in advanced details). This will configure the web server as the instances are provisioned.

```bash
#!/bin/bash
# Update all yum package repositories
yum update -y
# Install Apache Web Server
yum install -y httpd.x86_64
# Start and Enable Apache Web Server
systemctl start httpd.service
systemctl enable httpd.service
# Add our custom webpage HTML code to "index.html" file.
sudo echo "<h1>Welcome to My Website</h1>" > /var/www/html/index.html
```
After finishing the launch template, check that the subnets are correctly listed for the ASG, navigate to the Listeners and routing section, and for the default routing option, select "Create a new target group".

Clicking next will bring you to the Scaling Policies configuration for your ASG. Configuring your group sizes and capacity to match the following images will ensure that you have 2 EC2 instances running your web page at all times. The instances will be provisioned across your subnets, meaning the instances are multi-AZ and highly available. Setting the scaling policy to match the following pictures will cause more instances to be provisioned if your EC2 instances' CPU utilization goes over 50% - this will ensure that your architecture is highly available.

![](https://i.imgur.com/BQsUevX.png)

![](https://i.imgur.com/gF4sC0D.png)

Once you finish all the configuration for the Auto Scaling Group, your instances should automatically begin to provision themselves to meet the desired capacity (2). Once your instances are fully up and running, you can check the details of the instances and visit the public IPv4 address to check if your website is up and running.

![](https://i.imgur.com/GZCvPEO.png)
# Step 3: Application Tier
Configuration of the application tier is very similar to the web tier, although there are some key differences between the configuration of the two tiers. So starting from the top, we need to make a new security group. The `app-sg` will allow SSH and ICMP-IPv4 from the web tier only. Note that `sg-0f3d5980d71c85f73` is the security group `web-sg` that we created in step 2.

![](https://i.imgur.com/ORCVvAp.png)

The steps in configuring the ASG for the application tier need a bit of adjustment from step one, but overall it follows the same pattern.

In the launch template, most of the settings will be the same as the web tier. You need to use the `app-sg` you created for this launch template, and you can omit the user data script that configures the web server.

For the ASG for the application tier, select 2 private subnets that are in different AZs to ensure that this tier is just as highly available as the web tier.

You will need to create another application load balancer (ALB) for the application tier ASG. However, you need to make sure the ALB is "Internal" as we do not want the application tier to face the internet. Create a new target group, and use the same group size settings and scaling policy as the web tier.

After creating the ASG, you should have a total of 4 EC2 instances launched. You can SSH (I personally used PuTTY for this project, so I'll be using the .ppk key pair used in the launch configuration) to check the status of your infrastructure at this point. In PuTTY, navigate in the Category panel:

Connections -> SSH -> Auth -> Private key file (.ppk)

Return to Sessions in the Category panel, insert the public IPv4 of one of your web tier instances, and open to connect. Log in as `ec2-user` and send a ping to one of the application tier EC2 instances to test the configuration of your architecture so far.
```
ping -c 5 <privateIP>
```
![](https://i.imgur.com/cRIZ7rR.png)
![](https://i.imgur.com/GRBFhwN.png)
# Step 4: Database Tier
For this layer, we will need to provision a multi-AZ MySQL database using RDS. Before we begin creating and configuring the database, we first will set up a "DB subnet group" found on the RDS dashboard. Choose your VPC, select both AZs we are working with, and add the remaining two private subnets to the DB subnet group. We can refer to the DB subnet group as `dbsubnetgroup`

![](https://i.imgur.com/anETSAe.png)

We will begin configuring the RDS by selecting the Engine type "MySQL" and setting the template to "Dev/Test" (note: AWS Free-Tier does not offer multi-AZ deployment).

![](https://i.imgur.com/ZoEYnmd.png)

After you enable "Multi-AZ DB instance", scroll down to instance configuration and select "burstable classes" and use "db.t2.micro" for the instance class. Next, configure the master username and password for your database, then you can proceed to the connectivity settings.

For the connectivity settings use the following:
- Don't connect to EC2 compute resource
- VPC - The VPC used in Step 1
- Subnet Group - `dbsubnetgroup`
- Public Access - No
- Security Group - Create new, name it `db-sg`, and dont change any settings. We will adjust the rules after we launch our database.

![](https://i.imgur.com/0jep7vZ.png)

After creating the database, wait a few minutes for the multi-AZ instance to fully deploy. While you are waiting for the database to fully deploy, go to the security group settings and edit the inbound rules for `db-sg`. Remove the existing rule that is there by default, and replace it with a new rule.

Configure this new rule to allow "MySQL/Aurora" traffic on port 3306, and for the source use `app-sg`. This will ensure that your database is only getting connections from the application tier, further decoupling the database from the internet.

![](https://i.imgur.com/o3qigSJ.png)

# Step 5: Cleaning up the Route Tables
Once the web, application, and database tiers are all up and running, then your 3-tier archetecture is complete! You can now benefit from the scalability, reliability, and security that comes with using this architecture! There are still some additional tweaks and cleanup that I like to do that make the architecture nice and neat.

As you recall in step 1, using the "VPC and more" option caused AWS to automatically provision route tables when creating our VPC. This last and final step of configuring our 3-tier architecture will just focus on cleaning up the route tables to make configurations easier in the future.

We currently have 6 route tables in total inside our VPC, the main route table does not show up on the resource map until after creation:
- 1 main route table (0 subnet associations)
- 1 public route table (2 public subnet associations)
- 4 private route tables (1 private subnet association each)

![](https://i.imgur.com/kPN1Ein.png)

You can see by the following picture that the public route table is already configured how we want, as both public subnets are associated with it as expected. The private route tables have one subnet association each, so we will be consolidating them by tier. You can make this public route table the main table, and follow that up by deleting the old main table in your VPC. Just note unassociated subnets will default to whatever the main route table is when provisioned, so exercise caution when adding to this architecture.

![](https://i.imgur.com/tljfrAI.png)

We can pick one of the two private route tables that is associated with a subnet hosting the application tier instances and delete it. We can then associate the subnet that was on the table we deleted with the private routing table associated with the other subnet hosting the application tier instances. This will leave us with one private routing table for the two subnets hosting the application tier instances. You can repeat the same process of consolidating the route tables for the two private routing tables for database tier subnets.

![](https://i.imgur.com/8eEdIgy.png)

Your final route table should be as follows, and your VPC resource map should look like this:
- 1 public route table - web tier (2 public subnet associations)
- 1 private route table - application tier (2 private subnet associations each)
- 1 private route table - database tier (2 private subnet associations each)

![](https://i.imgur.com/xbg6FXa.png)

# Conclusion

This highly available and scalable architecture is a proven pattern for infrastructure, as it highlights some of the key benefits of using cloud solutions. By utilizing the 3-tier structure, resources are effectively decoupled, ensuring flexibility and scalability. The auto scaling instances guarantee high availability and efficient scaling, while the load balancers further enhance the availability of the infrastructure. Leveraging multi-AZ deployment eliminates single points of failure and complements the use of ASG, ALB, and RDS read replicas. The implementation of security groups enables granular access control, ensuring the protection of our application and database while allowing authorized traffic. Overall, this project provides an excellent opportunity to work with complex infrastructure commonly utilized in professional environments.

Thank you for taking the time to explore this breakdown of the 3-tier architecture in AWS. I appreciate your continued interest and support. I encourage you to visit my website or explore other project breakdowns available on GitHub. Once again, thank you for your engagement, and I look forward to sharing more valuable content with you in the future.

⚠️ **Reminder: If you were following along and provisioned these resources on your personal AWS account, please be aware that some charges will be incurred if you do not properly remove them. I personally had access to a lab setting, so I had access to features outside of the AWS Free-Tier.** ⚠️
