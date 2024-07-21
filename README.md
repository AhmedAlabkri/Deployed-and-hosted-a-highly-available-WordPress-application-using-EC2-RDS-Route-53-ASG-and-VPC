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
     - Note, I could alternatively attach the AdministratorAccess policy which will grants full access to all AWS services and resources, but I wanted to have granular control.

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

  # AZ1 subnets:
  - To create the subnets for AZ1: Subnets -> Create subnet
  ![image](https://github.com/user-attachments/assets/b8ca49ea-d767-48a7-8014-bada1768c915)
  - Notes:
    - Dont forget to select the same AZ when creating the subnets.
    - Avoid overlapping the subnets IP ranges.
  # AZ2 subnets:
  ![image](https://github.com/user-attachments/assets/2a2037e8-bba1-4c64-bfc1-d4192a742f27)

