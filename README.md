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

# 9- Create the EC2 Instance Connect Endpoint
With an EC2 Instance Connect Endpoint (EIC), you can connect to instances within your VPC without needing SSH, a public IP address, or a bastion host.

![image](https://github.com/user-attachments/assets/74dfadd6-2322-4cf8-922d-e55237b43c9f)
- Note: Its important to place this endpoint in a private subnet.

  # Launch an EC2 Instance - To test whether we can connect to it through the EICE
  - Notes:
      - Select proceed without key pair, since we want to test the EICE.
      - Place it in any private subnet, similar to the EICE.
      - For the security group, select the Web server security group we created in step 8.
  # To test:
  - Select the EC2 instance we just created -> Connect -> Connect using EC2 Instance Connect Endpoint. And select the EICE and connect.
      - I got an error:
    ![image](https://github.com/user-attachments/assets/92023327-52f5-4bcd-9f94-f2576df7daed)

  - From the error message, it was clear that the problem was related to the security group configuration that we attached to it which is the EICE security group. To resolve this, I decided to inspect the security group settings.
  - It turned out that the problem was the outbound rule I created, for the CIDR range I accidentally used the Ip range: 0.0.0.0/16 instead of 10.0.0.0/16(The Ip I should use is the one I created my VPC with, you can find that in the column IPv4 CIDR in your VPC):
    ![image](https://github.com/user-attachments/assets/d18b8c73-8383-4328-8956-4afd19e62ffd)

  - Now after we solved the problem, we can run the following command to test: sudo yum update -y
    - The output means, there is no package to install currently. This mean the test was successful.
    ![image](https://github.com/user-attachments/assets/790b952d-d9c6-47f3-b371-a8221b55291f)

  # Test the EC2 instance Using AWS CLI
  - To test use this command in the terminal: aws ec2-instance-connect ssh --instance-id <Put the instance Id>
    - The test was successful too:
    ![image](https://github.com/user-attachments/assets/d364c55e-6678-4444-9824-2f3de7ef291c)

  - Using the command: sudo yum install nano -y we can test whether we can download packages in the EC2 instance or not. This command will install the text editor nano.
    - The test was successful:
    ![image](https://github.com/user-attachments/assets/fd64c473-460c-45d5-8db3-0325f7b4012f)
  - We can also test if we can download the Apache server: sudo yum install httpd -y

  - ***Now we have confirmed that we can SSH to our EC2 instance using the EICE from both the managment console and from the CLI command.***
  - The last thing we need to do here, is to terminate the EC2 after we done with the testing...

# 10- Create EFS
EFS Allows multiple EC2 instances to access the file system concurrently, making it ideal for scenarios that require shared storage, such as web serving, which is our use case here. In this case if we used EBS instead, any change we make to one EC2 instance would not be reflected in the other EC2 instances. leading to inconsistency in the shared data.

This is why we will store our wordpress code in the EFS, so that any change will be made to it, will be reflected in all EC2 instances.

- Notes:
    - Store the mount in the private data subnet for both AZ.
    - For the mount, make sure to select the EFS security group.
    - The mount target is essentially setting up a gateway for the EC2 instances to connect to the EFS. In our case, we will mount this EFS to our servers.
 
# 11- Create RDS Instance - In the private data subnet

 # Subnet Group
    Before we create the RDS, we want to create the subnet group. This way we can specifiy which subnet we want to create our database in.
    - Notes:
        - Under **add subnet** in the subnet group creation process, we can specifiy the subnets that we want to create our DB in. We will 
     select the private data subnet.
        ![image](https://github.com/user-attachments/assets/226944ba-7829-44b3-9ee4-a9787b4d82f4)
 # RDS
    - Notes:
        - Save the master name and password somewhere before continuing.
    ![image](https://github.com/user-attachments/assets/bb60d32c-2836-4d45-a2a6-ec96a6b381b0)

# 12- Load Balancer
We will place our server (EC2 instance) in the private subnet to increase security. Therefore, we need to create a target group to conncet the load balancer to this EC2 instance. 
So we will do these steps:
- EC2 instance placed in private subnet.
- Target group and register the EC2 instance into it.
- Create the application load balancer in the public subnet.
- Configure the application load balancer to use the target group.
# Create Target group:
![image](https://github.com/user-attachments/assets/bc557252-2a87-44bc-bee9-64f6be0b901e)

# Create EC2 instance:
- Notes:
    - We will place this EC2 instance in the web private subnet AZ1, its also possible to place it in the web private subnet AZ2.
    - Since we are using the EC2 endpoint (that we created in step #9), we dont need a key pair when creating the EC2.
    - Make sure to select the correct security group (the web server security group).
 ![image](https://github.com/user-attachments/assets/936eca3a-8689-446e-a11a-9ecec3eecad5)

# Register EC2 instance to the Target Group:
- To do this: EC2 -> Target groups(select the target group we just created) -> Register targets(select the EC2 instance).
![image](https://github.com/user-attachments/assets/50927d44-369d-4bc5-a644-557ecc1f4a26)

# Create the Application Load Balancer:
- From: EC2 -> Load balancers -> ALB
- Notes:
    - ALBs are designed to handle HTTP and HTTPS traffic, which is ideal for our web application
    - In the Network mapping section, make sure to select the public subnet, an ALB is always asked to have the reach to a public subnet.
    - Select the ALB security group we created.
![image](https://github.com/user-attachments/assets/594c49bd-4b47-44ec-a8fb-10fa29c4c4a0)

# 13- Installing a Website on an EC2 Instance
In order to install any application in your server, you first need to find the correct documentation that gives you the installation steps and commands that you will need to run. In our case I made a google search to find these links:
- The installation proccess with the specific commands I will need:
https://docs.aws.amazon.com/linux/al2023/ug/hosting-wordpress-aml-2023.html
- Wordpress requirements:
https://wordpress.org/about/requirements/

# Install Wordpress:

# Setup WordPress on Amazon Linux 2023 (AL2023)

```sh
# Switch to root user
sudo su

# Update system packages
sudo dnf update -y

# Create HTML directory
sudo mkdir -p /var/www/html

# Set environment variable for EFS DNS name
EFS_DNS_NAME=

# Mount the EFS to the HTML directory
sudo mount -t nfs4 -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport "$EFS_DNS_NAME":/ /var/www/html

# Install and start Apache web server
sudo dnf install -y httpd
sudo systemctl enable httpd
sudo systemctl start httpd

# Install PHP 8 and necessary extensions for WordPress
sudo dnf install -y php php-cli php-cgi php-curl php-mbstring php-gd php-mysqlnd php-gettext php-json php-xml php-fpm php-intl php-zip php-bcmath php-ctype php-fileinfo php-openssl php-pdo php-tokenizer

# Install MySQL Community Repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server

# Start and enable MySQL server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# Set permissions
sudo usermod -a -G apache ec2-user
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www && find /var/www -type d -exec sudo chmod 2775 {} \;
sudo find /var/www -type f -exec sudo chmod 0664 {} \;
sudo chown apache:apache -R /var/www/html

# Download and extract WordPress files
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/html/

# Create the wp-config.php file
sudo cp /var/www/html/wp-config-sample.php /var/www/html/wp-config.php

# Edit the wp-config.php file to set database details
sudo nano /var/www/html/wp-config.php

# Restart the webserver
sudo systemctl restart httpd
```

- Notes:
    - Like we did in step #9, connect to the EC2 using the EICE, and run each command.
    - For  ```EFS_DNS_NAME=``` copy your EFS DNS name and paste it after the ```=```.
    - After running ```sudo nano /var/www/html/wp-config.php``` you will have to fill in your information:
```
define('DB_NAME', 'wordpress_db');
define('DB_USER', 'wordpress_user');
define('DB_PASSWORD', 'password');
define('DB_HOST', 'localhost');
```
    - From: EC2 -> Target groups -> Select your Target Group -> Then check that your Target group is healthy:
    
    ![image](https://github.com/user-attachments/assets/15fb2261-6985-48a6-b18a-0a5a7cf2ba2e)


# Open Wordpress:
    - From the Load balancer details, copy your DNS name and paste it in new browse page:
    
    ![image](https://github.com/user-attachments/assets/5f565819-4216-4db8-bb69-c7f191cb1749)
    - After Signning in:
    ![image](https://github.com/user-attachments/assets/89b01f92-a307-42e1-8690-a5066fb8beaa)

# 14- DNS Configuration
To Start the DNS configuration, we will first need to register our domain name in route 53.

# Register a New Domain Name in Route 53:

- Route 53 -> Registered domains -> Register domains

![image](https://github.com/user-attachments/assets/25baf4ad-7e34-40bb-83b2-9f085cfc54fd)

- The domain name under progress, you might get a Domain registration failed meassage, in this case make sure you turn on MFA for your account, for me this works..
- Usually it takes up to 15 minutes:
![image](https://github.com/user-attachments/assets/54c40aa5-d8a5-4f7b-9fd8-48aeb91644c2)

![image](https://github.com/user-attachments/assets/83cdcc9b-778f-4188-b284-97795d82be1e)



# Creating a record set:
When you create a domain name, it is not automatically attached to your load balancer. You need to explicitly link your domain name to your load balancer's DNS name using a DNS record set.

- To create the record set: Route 53 -> Hosted zones -> your domain name -> Create record
- These are the settings and configurations:
  ![image](https://github.com/user-attachments/assets/66c019a4-eb5d-4828-b0d9-ac5949e22721)
    - Notes:
        - Pick the Region you created your Load balancer in, in my case it is US EAST OHIO.
        - Since we created a domain name, you should go to your admin page in wordpress and update your url to the domain name from the settings.
        - You should receive an email from AWS to verify your Domain name, if not verified, the domain name will be suspended.

# Creating SSL Certificate (Secure Sockets Layer):
Which is a digital certificate that encrypts data transmitted between a user's web browser and a website, ensuring that any information exchanged (like passwords or credit card numbers) is protected from eavesdroppers.

- To create: AWS Certificate Manager -> Certificates -> Request certificate -> Request public certificate
    - Notes:
        - You can click **Add another name to this certificate** and add a wild card, for example *.example.com
![image](https://github.com/user-attachments/assets/7864d584-ffcb-487e-865f-2830d3b5793c)
Now we have to create a record set in route 53 to validate that this domain name is belong to us, you can do that by clicking **Create records in Route 53**.

![image](https://github.com/user-attachments/assets/06b66b1e-6947-4bb8-b09d-e15c07a0256e)

# Create an HTTPS Listener:
Now we will use the SSL certificate to encrypt the data exchanged between the user and the website. In general SSL certificate is required to enable HTTPS listener. This is why we didnt create it earlier.
- Add listener: You can do that from the load balancer dashboard.
    - For the routing actions, select forward to target groups and select the dev target group that we created above.
    - Make sure you select **(from ACM)** in the **Default SSL/TLS server certificate** section.

![image](https://github.com/user-attachments/assets/b572dac0-1955-4c70-8a47-492eeb56d8b1)
- Notes:
    - Select the http listener and change its settings to **redirect to URL** and **FULL URL**. As you might know, the http is insecure.
    - This is also why we added port 443 in our Load balancer security group.
![image](https://github.com/user-attachments/assets/6f47b1d2-b557-4131-99ce-c7a0d596a93d)

# Modify WP CONFIG file:
We will add a code to the file that tells the server that we want to redirect from http to https.

```
/* SSL Settings */
define('FORCE_SSL_ADMIN', true);

// Get true SSL status from AWS load balancer
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
  $_SERVER['HTTPS'] = '1';
}
```
- I tried to explain each line of code here:

```
define('FORCE_SSL_ADMIN', true);
```

- Ensures all traffic to the WordPress admin area is encrypted over HTTPS, protecting sensitive data like login credentials from interception.

```
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
  $_SERVER['HTTPS'] = '1';}
```

- Checks if the request was initially made over HTTPS before reaching the load balancer and sets the connection as secure within WordPress to avoid mixed content issues.

- Notes:
    - Connect to the EICE and run the following commands:
        - ```sudo su```
        - ```cd /var/www/html/```
        - ```vi wp-config.php```
        Then add the above code:
![image](https://github.com/user-attachments/assets/65a8a90c-ad0a-4860-9518-4b073822efcf)
        - Dont forget to also run: ```service httpd restart``` after you update your server, to restart the apache service.
