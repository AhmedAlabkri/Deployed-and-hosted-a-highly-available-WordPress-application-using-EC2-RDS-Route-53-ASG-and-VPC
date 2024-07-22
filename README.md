# Deployed-and-hosted-a-highly-available-WordPress-application-using-EC2-RDS-Route-53-ASG-and-VPC
In the below steps, sometimes I typed some notes that I thought were useful when I was building this project...
# Steps

# 1- IAM user
- Firstly I created an IAM user for myself, to enhance security. Its usually better to avoid using the root user account.
- To access the IAM page, just type IAM in the search bar on the AWS console.
-  For the polices, these are the list of polices that I attached to my IAM account:
    - AmazonEC2FullAccess
    - AmazonRDSFullAccess
    - AmazonRoute53FullAccess
    - AmazonVPCFullAccess
    - AutoScalingFullAccess
    - CloudWatchFullAccess
    - ElasticLoadBalancingFullAccess
    - AmazonRoute53ResolverFullAccess
    - GetResourceShareAssociations 
   
   I think these are good enough to do the project, later I can add more polices if I needed to...
     - Note, I could alternatively attach the AdministratorAccess policy which will grants full access to all AWS services and resources and this is generally the best approach, but I wanted to learn more granular polices, so every time I needed new policy I will just add it without mentioning that in this document.

  - Now I can log in to the IAM account and work from there.

# 2-  Generate Access Key for the IAM User
- The access key is important. This is because the access key is used for programmatic access via the AWS CLI or API calls.
- From the IAM page: Users -> Select the user -> Security credentials -> Create access key -> Command Line Interface (CLI).
  - Currently we need to use the CLI this is why we select it.

# 3- Configure the access key
![Screenshot 2024-07-20 190621](https://github.com/user-attachments/assets/cf39b793-7bab-4fa9-9d15-e731ffa768b9)
- To configure the access key that we just created, in the terminal:
  - aws configure
  - Then type the access key and the secret access key
  - Some notes: 
    - For the output format there are three different formats:
        1- Table 2- JSON  3- text
      The default is JSON, so by just pressing enter, the output is going to be in JSON format, and this is what we will do.
    
    - C:\Users\username\.aws\ is the path to find the credintals and configuration info that we just entered..
   
    - Its important to not share the acccess key or secret access key for security reasons. For this project, I did so because I will delete this IAM user and the access key before I publish the document.

# 4- Testing the configuration
I ran the command: aws iam get-user. (This command retrieves information about the specified IAM user, or if not specified it will give the information about this user). But I got an error.
![image](https://github.com/user-attachments/assets/e5de9b10-6d57-4bf4-b693-37e196099830)

 This error is because when I tried to be granular with giving permissions to the IAM-user(in the first step), I forgot to attach the IAMReadOnlyAccess policy, so I will do that now.

![image](https://github.com/user-attachments/assets/97f8c63d-cf10-4f9c-96c3-e14695cb4174)
- Now after attaching the IAMReadOnlyAccess policy, we have read-only access to IAM resources. So when we try the "aws iam get-user" command it works as you can see above.

# 5- Creating VPC
- Select the region where you want to create the VPC, in my case its going to be us-east-2
- Then in the search bar type VPC and enter -> Create VPC -> VPC only
  - I selected VPC only, because I will customize the network layout, including specific subnets, route tables, and gateways. Otherwise I would select VPC and more. for the default configurations.
- Settings used to create the VPC:
  -  IPv4 CIDR manual input: 10.0.0.0/16. This mean the VPC can have IP addresses from 10.0.0.0 to 10.0.255.255
  -  No IPv6 CIDR block
  -  Tenancy: Default
  -  Create VPC.

# 6- Enable DNS Host Name Setting in VPC & Create internet gateway
- The benefits of enabling the DNS host name: a) easier to identify instances by name rather than by IP address. b) Instances with public IP addresses will have corresponding public DNS names, which can be used to access the instances over the internet.
  - To enable: VPC -> Your VPCs -> select the vpc -> Edit VPC settings. And then select Enable DNS hostnames.
    
- Benefit of internet gateway: To enable instances (like EC2) in VPC to connect to the internet.
  - To create the internet gateway: VPC -> Internet gateways -> Create internet gateway.
    - After that select attach to VPC to attach it to the same VPC we created above.
      ![Screenshot 2024-07-21 002137](https://github.com/user-attachments/assets/1736bc28-3ae6-47fc-aacc-b8ef61e1322b)

    - Note, for each VPC, you can only attach one Internet gateway.

# 7- Subnets
- Creating subnets within a VPC allows to segment the network into public and private subnets.
  - Public subnet: We store things that will likely be accessed from the internet like the EC2 instances. Also NAT Gateway, to allow instances in private subnets to access the internet while remaining inaccessible from the internet
  - Private subnet: Things that should not be accessed from the internet. For example, databases (RDS).

  # Creating AZ1 subnets:
  - To create the subnets for AZ1: Subnets -> Create subnet
  ![image](https://github.com/user-attachments/assets/b8ca49ea-d767-48a7-8014-bada1768c915)
  - Notes:
    - Dont forget to select the same AZ when creating the subnets.
    - Avoid overlapping the subnets IP ranges.
  # Creating AZ2 subnets:
  ![image](https://github.com/user-attachments/assets/2a2037e8-bba1-4c64-bfc1-d4192a742f27)
  # Auto-Assign IP in Public Subnets:
  When we enable the auto-assing IP for each public subnet in the VPC, each resource we launch in the subnet will automatically assigned a public IP. Which  simplifies the process of ensuring instances have internet access.
  - To enable: VPC -> Pick the public subnet -> from actions select Edit subnet -> Enable auto-assign public IPv4 address.
  - I did that for both public subnets.
  # Public Subnets Route Table:
  To make the public subnets public, we need to create route tables that have a route to the internet.
  - To create the route table: VPC -> Route tables -> Create route table
  - Then you can select the route table and click edit routes to add the route to the internet which is 0.0.0.0/0
      - Notes:
          - For the destination: 0.0.0.0/0 (Internet route). And for the target: Your IGW.
          - Now any subnet will be public, when we attach it to this route table.
  # Private subnets:
  When you dont explicitly associate a subnet to a route table, this subnet will automatically be associated with the main route table. This main route table is local. In other words, a subnet is private by default since its associated with this local & main route table.

    - As you can see in the below image, we have two route tables even though we only created one...
  ![image](https://github.com/user-attachments/assets/fb48bf35-53e3-49ca-b9c1-52694a70bc8d)

    - In the **subnets without explicit associations** you can see all the subnets that we didnt associate with a route table are associated with this main route table:
  ![image](https://github.com/user-attachments/assets/2d34907c-0650-45d9-b117-4dd32d400ad9)

 # Setting up the NAT Gateway & Route table for the private subnets
  Setting up a nat gateway is important to allow instances in private subnets to access the internet while remaining inaccessible from the internet. So we going to 
 put the NAT Gateway in the public subnet as discussed earlier. Then we will create a route table that connect the private subnets to this NAT Gateway.
 - To set it up: VPC -> NAT gateways -> Create NAT gateway. Connectivity type should be public, and Allocate Elastic IP.
 - After creating the NAT Gateway, create a routing table in the edit routes add the destination and target, this time the target is the NAT gateway we just created:
   ![image](https://github.com/user-attachments/assets/323ef060-5a91-4340-a0a4-a8d020a8937c)

 - Notes:
     - The NAT Gateway is always placed in the public subnet.
     - Its best practice to create a NAT Gateway for each AZ, however for this project and to minimize cost, I only created one for all subnets.
     - Dont forget to associate the private subnets to the route table we just created.

# 8- Security Groups
Security groups are the firewalls for our VPC. We will create 5 different security groups as follows:
- Security group for the load balancer
    - For http and https - ports 80 and 443.
- Security group for the EC2
    - Traffic coming from the CIDR of the VPC - port 22.
- Security group for the web server
    - http and https from the load balancer security group only.
    - ssh (port 22) only if its coming from the EC2 security group.
- Security group for the database (the RDS)
    - Traffic from port 3306 if its coming from the web server.
- Security group for EFS
    - Traffic from port 2049 only if its coming from the web server.
    - Traffic from port 22 if its coming from the EC2 endpoint security group. 

![image](https://github.com/user-attachments/assets/55c11039-410a-4d60-8e57-27025ff33262)


