# Deployed-and-hosted-a-highly-available-WordPress-application-using-EC2-RDS-Route-53-ASG-and-VPC
This file provides a comprehensive step-by-step walkthrough for deploying and hosting a highly available WordPress application using AWS services such as EC2, RDS, Route 53, Auto Scaling Groups (ASG), and Virtual Private Cloud (VPC). This guide includes detailed instructions, screenshots, and insights gained during the project to help others replicate the setup and understand the deployment process.

For an introduction to the project and an overview of its main components, please visit the [README.md](./README.md) file.

Here's your text with improved clarity, consistent indentation and spacing, and corrected grammar:

---

In the steps below, I have included notes that I found useful while building this project.

# Steps

## 1. IAM User

- First, I created an IAM user for myself to enhance security. It is usually better to avoid using the root user account.
- To access the IAM page, type **IAM** in the search bar on the AWS console.
- For the policies, the only policy we need to attach to the IAM account is the **AdministratorAccess** policy, which grants full access to all AWS services and resources.
- Once the IAM user is created, I can log in to the IAM account and work from there.

## 2. Generate Access Key for the IAM User

- The access key is important because it is used for programmatic access via the AWS CLI or API calls.
- From the IAM page: **Users -> Select the user -> Security credentials -> Create access key -> Command Line Interface (CLI)**.
- We select CLI because we currently need to use the command line interface.

## 3. Configure the Access Key

![Screenshot 2024-07-20 190621](https://github.com/user-attachments/assets/cf39b793-7bab-4fa9-9d15-e731ffa768b9)

- To configure the access key that we just created, open the terminal and run:
  - `aws configure`
  - Enter the access key and the secret access key.
  
  - **Some notes:**
    - For the output format, there are three different formats:
      1. Table
      2. JSON
      3. Text
    - The default is JSON, so by pressing Enter, the output will be in JSON format, which is what we will use.
    - `C:\Users\username\.aws\` is the path to find the credentials and configuration information that we just entered.
    - It is important not to share the access key or secret access key for security reasons. For this project, I will delete this IAM user and the access key before publishing the document.

## 4. Testing the Configuration

I ran the command: `aws iam get-user`. This command retrieves information about the specified IAM user, or if not specified, it provides information about the current user. However, I encountered an error.

![image](https://github.com/user-attachments/assets/e5de9b10-6d57-4bf4-b693-37e196099830)

Initially, I avoided using the **AdministratorAccess** policy to try a more granular permissions approach. However, I realized that this required multiple adjustments to identify the necessary policies, so I deleted all permissions and used **AdministratorAccess** for full access.

![image](https://github.com/user-attachments/assets/97f8c63d-cf10-4f9c-96c3-e14695cb4174)

- After attaching the **AdministratorAccess** policy, I had full access to AWS services, and the `aws iam get-user` command worked as expected.

## 5. Creating a VPC

- Select the region where you want to create the VPC. In my case, it is **us-east-2**.
- Then, in the search bar, type **VPC** and enter: **Create VPC -> VPC only**.
  - I selected **VPC only** because I wanted to customize the network layout, including specific subnets, route tables, and gateways. Otherwise, I would select **VPC and more** for default configurations.

- **Settings used to create the VPC:**
  - **IPv4 CIDR manual input:** `10.0.0.0/16`. This means the VPC can have IP addresses from `10.0.0.0` to `10.0.255.255`.
  - **No IPv6 CIDR block**
  - **Tenancy:** Default
  - **Create VPC**

## 6. Enable DNS Hostname Setting in VPC & Create Internet Gateway

- **Benefits of enabling the DNS hostname:**
  - Easier to identify instances by name rather than by IP address.
  - Instances with public IP addresses will have corresponding public DNS names, which can be used to access the instances over the Internet.

  - To enable: **VPC -> Your VPCs -> select the VPC -> Edit VPC settings**, and then select **Enable DNS hostnames**.
    
- **Benefits of the Internet gateway:** To enable instances (like EC2) in the VPC to connect to the Internet.

  - To create the Internet gateway: **VPC -> Internet gateways -> Create Internet gateway**.
    - After that, select **Attach to VPC** to attach it to the same VPC we created above.

![Screenshot 2024-07-21 002137](https://github.com/user-attachments/assets/1736bc28-3ae6-47fc-aacc-b8ef61e1322b)

- **Note:** For each VPC, you can only attach one Internet gateway.

## 7. Subnets

- Creating subnets within a VPC allows for segmentation of the network into public and private subnets.
  - **Public subnet:** Used for resources that need to be accessed from the Internet, such as EC2 instances. It also contains the NAT Gateway to allow instances in private subnets to access the Internet while remaining inaccessible from the Internet.
  - **Private subnet:** Used for resources that should not be accessed from the Internet, such as databases (RDS).

### Creating AZ1 Subnets:

- To create the subnets for AZ1: **Subnets -> Create subnet**.

![image](https://github.com/user-attachments/assets/b8ca49ea-d767-48a7-8014-bada1768c915)

- **Notes:**
  - Don't forget to select the same AZ when creating the subnets.
  - Avoid overlapping the subnets' IP ranges.

### Creating AZ2 Subnets:

![image](https://github.com/user-attachments/assets/2a2037e8-bba1-4c64-bfc1-d4192a742f27)

### Auto-Assign IP in Public Subnets:

When we enable auto-assign IP for each public subnet in the VPC, each resource launched in the subnet will automatically be assigned a public IP. This simplifies the process of ensuring instances have Internet access.

- To enable: **VPC -> Pick the public subnet -> Actions -> Edit subnet -> Enable auto-assign public IPv4 address**.
- I did this for both public subnets.

### Public Subnets Route Table:

To make the public subnets public, we need to create route tables that have a route to the Internet.

- To create the route table: **VPC -> Route tables -> Create route table**.
- Select the route table and click **Edit routes** to add the route to the Internet, which is `0.0.0.0/0`.

  - **Notes:**
    - For the destination: `0.0.0.0/0` (Internet route). For the target: Your IGW (Internet Gateway).
    - Any subnet attached to this route table will be public.

### Private Subnets:

When you don't explicitly associate a subnet with a route table, it automatically associates with the main route table, which is local. This means a subnet is private by default since it's associated with the main route table.

- As shown in the image below, we have two route tables even though we only created one.

![image](https://github.com/user-attachments/assets/fb48bf35-53e3-49ca-b9c1-52694a70bc8d)

- In the **subnets without explicit associations**, you can see all the subnets that aren't associated with a route table are associated with this main route table.

![image](https://github.com/user-attachments/assets/2d34907c-0650-45d9-b117-4dd32d400ad9)

### Setting up the NAT Gateway & Route Table for Private Subnets:

Setting up a NAT Gateway is important to allow instances in private subnets to access the Internet while remaining inaccessible from the Internet. We'll place the NAT Gateway in the public subnet as discussed earlier. Then, we'll create a route table that connects the private subnets to this NAT Gateway.

- To set it up: **VPC -> NAT gateways -> Create NAT gateway**. Connectivity type should be public, and allocate an Elastic IP.
- After creating the NAT Gateway, create a routing table and, in **Edit routes**, add the destination and target; this time, the target is the NAT gateway we just created.

![image](https://github.com/user-attachments/assets/323ef060-5a91-4340-a0a4-a8d020a8937c)

- **Notes:**
  - The NAT Gateway is always placed in the public subnet.
  - It's best practice to create a NAT Gateway for each AZ. However, for this project and to minimize costs, I only created one for all subnets.
  - Don't forget to associate the private subnets with the route table we just created.

## 8. Security Groups

Security groups act as firewalls for our VPC. We will create five different security groups as follows:

- **Security group for the load balancer:**
  - For HTTP and HTTPS - ports 80 and 443.

- **Security group for the EC2:**
  - Traffic coming from the CIDR of the VPC - port 22.

- **Security group for the web server:**
  - HTTP and HTTPS from the load balancer security group only.
  - SSH (port 22) only if it's coming from the EC2 security group.

- **Security group for the database (the RDS):**
  - Traffic from port 3306 if it's coming from the web server.

- **Security group for EFS:**
  - Traffic from port 2049 only if it's coming from the web server.
  - Traffic from port 22 if it's coming from the EC2 endpoint security group.

![image](https://github.com/user-attachments/assets/55c11039-410a-4d60-8e57-27025ff33262)

## 9. Create the EC2 Instance Connect Endpoint

With an EC2 Instance Connect Endpoint (EIC), you can connect to instances within your VPC without needing SSH, a public IP address, or a bastion host.

![image](https://github.com/user-attachments/assets/74dfadd6-2322-4cf8-922d-e55237b43c9f)

- **Note:** It's important to place this endpoint in a private subnet.

### Launch an EC2 Instance - To test connectivity through the EICE

- **Notes:**
  - Select "proceed without key pair" since we want to test the EICE.
  - Place the instance in any private subnet, similar to the EICE.
  - For the security group, select the Web server security group we created in step 8.

### To Test:

- Select the EC2 instance we just created -> Connect -> Connect using EC2 Instance Connect Endpoint. Then select the EICE and connect.
  
  - I encountered an error:
  
    ![image](https://github.com/user-attachments/assets/92023327-52f5-4bcd-9f94-f2576df7daed)

  - From the error message, it was clear that the problem was related to the security group configuration attached to it, which was the EICE security group. To resolve this, I inspected the security group settings.
  
  - The issue was with the outbound rule I created; I accidentally used the IP range `0.0.0.0/16` instead of `10.0.0.0/16` (the IP I should use is the one I created my VPC with, found in the IPv4 CIDR column in your VPC):
  
    ![image](https://github.com/user-attachments/assets/d18b8c73-8383-4328-8956-4afd19e62ffd)

  - After resolving the issue, we can run the following command to test: `sudo yum update -y`
    
    - The output indicates there is no package to install currently. This means the test was successful.
    
    ![image](https://github.com/user-attachments/assets/790b952d-d9c6-47f3-b371-a8221b55291f)

### Test the EC2 Instance Using AWS CLI

- To test, use this command in the terminal: `aws ec2-instance-connect ssh --instance-id <Put the instance Id>`
  
  - The test was successful too:
  
    ![image](https://github.com/user-attachments/assets/d364c55e-6678-4444-9824-2f3de7ef291c)

- Using the command: `sudo yum install nano -y`, we can test whether we can download packages on the EC2 instance. This command will install the text editor `nano`.
  
  - The test was successful:
  
    ![image](https://github.com/user-attachments/assets/fd64c473-460c-45d5-8db3-0325f7b4012f)

- We can also test if we can download the Apache server: `sudo yum install httpd -y`

- **Conclusion:** We have confirmed that we can SSH into our EC2 instance using the EICE from both the management console and the CLI command.

- The last step here is to terminate the EC2 instance after we're done with testing.

## 10. Create EFS

EFS allows multiple EC2 instances to access the file system concurrently, making it ideal for scenarios that require shared storage, such as web serving, which is our use case here. If we used EBS instead, any change we make to one EC2 instance would not be reflected in the other EC2 instances, leading to inconsistency in the shared data.

This is why we will store our WordPress code in the EFS so that any change made to it will be reflected in all EC2 instances.

- **Notes:**
  - Store the mount in the private data subnet for both AZs.
  - For the mount, ensure to select the EFS security group.
  - The mount target essentially sets up a gateway for the EC2 instances to connect to the EFS. In our case, we will mount this EFS to our servers.

## 11. Create RDS Instance - In the Private Data Subnet

### Subnet Group

Before we create the RDS, we need to create the subnet group. This way, we can specify which subnet we want to create our database in.

- **Notes:**
  - Under **add subnet** in the subnet group creation process, we can specify the subnets where we want to create our DB. We will select the private data subnet.

    ![image](https://github.com/user-attachments/assets/226944ba-7829-44b3-9ee4-a9787b4d82f4)

### RDS

- **Notes:**
  - Save the master name and password somewhere before continuing.

  ![image](https://github.com/user-attachments/assets/bb60d32c-2836-4d45-a2a6-ec96a6b381b0)

## 12. Load Balancer

We will place our server (EC2 instance) in the private subnet to increase security. Therefore, we need to create a target group to connect the load balancer to this EC2 instance. We will follow these steps:

- Place the EC2 instance in the private subnet.
- Create a target group and register the EC2 instance into it.
- Create the application load balancer in the public subnet.
- Configure the application load balancer to use the target group.

### Create Target Group

![image](https://github.com/user-attachments/assets/bc557252-2a87-44bc-bee9-64f6be0b901e)

### Create EC2 Instance

- **Notes:**
  - Place this EC2 instance in the web private subnet AZ1. It's also possible to place it in the web private subnet AZ2.
  - Since we are using the EC2 endpoint (created in step #9), we don't need a key pair when creating the EC2.
  - Ensure to select the correct security group (the web server security group).

  ![image](https://github.com/user-attachments/assets/936eca3a-8689-446e-a11a-9ecec3eecad5)

### Register EC2 Instance to the Target Group

- To do this: Navigate to **EC2 -> Target groups** (select the target group we just created) -> Register targets (select the EC2 instance).

  ![image](https://github.com/user-attachments/assets/50927d44-369d-4bc5-a644-557ecc1f4a26)

### Create the Application Load Balancer

- From: **EC2 -> Load balancers -> ALB**
- **Notes:**
  - ALBs are designed to handle HTTP and HTTPS traffic, which is ideal for our web application.
  - In the Network mapping section, ensure to select the public subnet. An ALB always needs access to a public subnet.
  - Select the ALB security group we created.

  ![image](https://github.com/user-attachments/assets/594c49bd-4b47-44ec-a8fb-10fa29c4c4a0)

## 13. Installing a Website on an EC2 Instance

To install any application on your server, you first need to find the correct documentation that provides the installation steps and commands required. In our case, I performed a Google search to find these links:

- The installation process with the specific commands needed:
  - [AWS Documentation for WordPress on Amazon Linux 2023](https://docs.aws.amazon.com/linux/al2023/ug/hosting-wordpress-aml-2023.html)
- WordPress requirements:
  - [WordPress Requirements](https://wordpress.org/about/requirements/)

### Install WordPress

#### Setup WordPress on Amazon Linux 2023 (AL2023)

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

- **Notes:**
  - Like we did in step #9, connect to the EC2 using the EICE, and run each command.
  - For `EFS_DNS_NAME=`, copy your EFS DNS name and paste it after the `=`.
  - After running `sudo nano /var/www/html/wp-config.php`, you will have to fill in your information:

  ```php
  define('DB_NAME', 'wordpress_db');
  define('DB_USER', 'wordpress_user');
  define('DB_PASSWORD', 'password');
  define('DB_HOST', 'localhost');
  ```

  - From: **EC2 -> Target groups -> Select your Target Group -> Then check that your Target group is healthy:**

    ![image](https://github.com/user-attachments/assets/15fb2261-6985-48a6-b18a-0a5a7cf2ba2e)

- **Notes:**

- Initially, the target group wasn't healthy for me, so I did the following to resolve the issue:
  - If you encounter an unhealthy target check with a 302 error, it might be due to the health check path being incorrectly set.

  - **Solution:** 
    1. Go to the target groups page -> Health checks -> Edit the path.
    2. Copy and paste this into the path: `/wp-admin/install.php`.
    3. Save the changes. It will take between 1-5 minutes for the health check to update.

  - **Explanation:** The health check was initially set to the root path (`/`), which caused a 302 redirect to `/wp-admin/install.php` because WordPress detected that the setup was not yet complete (you first need to complete an installation form to use Wordpress). By changing the health check path to `/wp-admin/install.php`, the load balancer could correctly verify the instance's health by directly accessing the installation page. 

  - If this solution doesn't resolve the issue, you may have a problem with your routing configuration or security groups. Double-check your settings to ensure that all configurations are correct.

### Open WordPress:

- From the Load balancer details, copy your DNS name and paste it in a new browser page:

  ![image](https://github.com/user-attachments/assets/5f565819-4216-4db8-bb69-c7f191cb1749)

- After signing in:

  ![image](https://github.com/user-attachments/assets/89b01f92-a307-42e1-8690-a5066fb8beaa)

## 14. DNS Configuration

To start the DNS configuration, we will first need to register our domain name in Route 53.

### Register a New Domain Name in Route 53

- **Steps:** Route 53 -> Registered domains -> Register domains

  ![image](https://github.com/user-attachments/assets/25baf4ad-7e34-40bb-83b2-9f085cfc54fd)

- The domain name registration is in progress. You might get a **Domain registration failed** message. If this happens, ensure that you have Multi-Factor Authentication (MFA) turned on for your account; this worked for me.
- The process usually takes up to 15 minutes:

  ![image](https://github.com/user-attachments/assets/54c40aa5-d8a5-4f7b-9fd8-48aeb91644c2)

  ![image](https://github.com/user-attachments/assets/83cdcc9b-778f-4188-b284-97795d82be1e)

### Creating a Record Set

When you create a domain name, it is not automatically attached to your load balancer. You need to explicitly link your domain name to your load balancer's DNS name using a DNS record set.

- **To create the record set:** Route 53 -> Hosted zones -> Your domain name -> Create record
- **Settings and configurations:**

  ![image](https://github.com/user-attachments/assets/66c019a4-eb5d-4828-b0d9-ac5949e22721)

  - **Notes:**
    - Pick the Region where you created your Load balancer. In my case, it is US EAST OHIO.
    - Since we created a domain name, you should go to your admin page in WordPress and update your URL to the domain name from the settings.
    - You should receive an email from AWS to verify your domain name. If not verified, the domain name will be suspended.

### Creating SSL Certificate (Secure Sockets Layer)

An SSL certificate is a digital certificate that encrypts data transmitted between a user's web browser and a website, ensuring that any information exchanged (like passwords or credit card numbers) is protected from eavesdroppers.

- **To create:** AWS Certificate Manager -> Certificates -> Request certificate -> Request public certificate

  - **Notes:**
    - You can click **Add another name to this certificate** and add a wildcard, for example, `*.example.com`.

  ![image](https://github.com/user-attachments/assets/7864d584-ffcb-487e-865f-2830d3b5793c)

Now, we have to create a record set in Route 53 to validate that this domain name belongs to us. You can do that by clicking **Create records in Route 53**.

![image](https://github.com/user-attachments/assets/06b66b1e-6947-4bb8-b09d-e15c07a0256e)

### Create an HTTPS Listener

Now we will use the SSL certificate to encrypt the data exchanged between the user and the website. An SSL certificate is required to enable the HTTPS listener, which is why we didn't create it earlier.

- **Add listener:** You can do this from the load balancer dashboard.
  - For the routing actions, select **forward to target groups** and select the dev target group we created above.
  - Make sure you select **(from ACM)** in the **Default SSL/TLS server certificate** section.

![image](https://github.com/user-attachments/assets/b572dac0-1955-4c70-8a47-492eeb56d8b1)

- **Notes:**
  - Select the HTTP listener and change its settings to **redirect to URL** and **FULL URL**. With this setting, any request sent to HTTP will be redirected to HTTPS. As you may know, HTTP is insecure.
  - This is also why we added port 443 in our Load balancer security group.

![image](https://github.com/user-attachments/assets/6f47b1d2-b557-4131-99ce-c7a0d596a93d)

### Modify WP CONFIG File

We will add a code to the file that tells the server that we want to redirect from HTTP to HTTPS.

```php
/* SSL Settings */
define('FORCE_SSL_ADMIN', true);

// Get true SSL status from AWS load balancer
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
  $_SERVER['HTTPS'] = '1';
}
```

- **Explanation of the code:**

  ```php
  define('FORCE_SSL_ADMIN', true);
  ```

  - Ensures all traffic to the WordPress admin area is encrypted over HTTPS, protecting sensitive data like login credentials from interception.

  ```php
  if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = '1';
  }
  ```

  - Checks if the request was initially made over HTTPS before reaching the load balancer and sets the connection as secure within WordPress to avoid mixed content issues.

- **Notes:**
  - Connect to the EICE and run the following commands:

    ```bash
    sudo su
    cd /var/www/html/
    vi wp-config.php
    ```

  - Then add the above code:

    ![image](https://github.com/user-attachments/assets/65a8a90c-ad0a-4860-9518-4b073822efcf)

  - Don't forget to also run: `service httpd restart` after you update your server, to restart the Apache service.

- Even though I only typed the domain name in the URL, the server directed me to the HTTPS port:

  ![image](https://github.com/user-attachments/assets/dc62f492-99c5-4827-b563-65fd71d27b8b)

## 15. Availability and Tolerance - Auto Scaling Group

To increase availability, we will create an Auto Scaling group that will automatically add or remove servers as needed. This ensures that our application can handle varying traffic loads and remains resilient against failures. Every time a new EC2 instance is added, it will automatically mount the EFS, as it contains the WordPress code. For more details about EFS, refer to step #10. This is the plan we will follow for this section:

1. Terminate the EC2 instance that we created manually. We will do this because we want the auto-scaling group to create our instance.
2. Create a shell script for the Launch template.
3. Create a Launch Template with all necessary EC2 configurations.
4. Create an Auto Scaling group.

### Termination of EC2

- **Steps:** EC2 -> Instances -> Select the EC2 instance -> Instance state -> Terminate.

### Create Shell Script

Recall from the WordPress documentation URL in step #13, the requirements for installing WordPress are: PHP version 7.4 or greater, MySQL version 8.0 or greater, OR MariaDB version 10.5 or greater, and HTTPS support. Therefore, this shell script will be prepared with these dependencies as well as the mounting to the EFS:

```bash
#!/bin/bash

# update the software packages on the ec2 instance 
sudo yum update -y

# install the apache web server, enable it to start on boot, and then start the server immediately
sudo yum install -y httpd
sudo systemctl enable httpd 
sudo systemctl start httpd

# install php 8 along with several necessary extensions for wordpress to run
sudo dnf install -y \
php \
php-cli \
php-cgi \
php-curl \
php-mbstring \
php-gd \
php-mysqlnd \
php-gettext \
php-json \
php-xml \
php-fpm \
php-intl \
php-zip \
php-bcmath \
php-ctype \
php-fileinfo \
php-openssl \
php-pdo \
php-tokenizer

# install the mysql version 8 community repository
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm 

# install the mysql server
sudo dnf install -y mysql80-community-release-el9-1.noarch.rpm 
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023
sudo dnf repolist enabled | grep "mysql.*-community.*"
sudo dnf install -y mysql-community-server 

# start and enable the mysql server
sudo systemctl start mysqld
sudo systemctl enable mysqld

# environment variable
EFS_DNS_NAME=

# mount the efs to the html directory 
echo "$EFS_DNS_NAME:/ /var/www/html nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 0 0" >> /etc/fstab
mount -a

# set permissions
chown apache:apache -R /var/www/html

# restart the webserver
sudo service httpd restart
```

### Create Launch Template

- **Steps:** EC2 -> Launch template

- **Notes:**
  - Select the **Auto Scaling guidance** checkbox.
  - Under the **Advanced details** section, in the **User data** box, paste your script.

- Don't forget to update the EFS DNS name in the script. You can find it in your EFS configuration.

  ![image](https://github.com/user-attachments/assets/fc849f2c-170c-44a7-89a2-32f1b99533a9)

### Create Auto Scaling Group

- **Steps:** EC2 -> Auto Scaling groups

- **Notes:**
  - Select the private web subnet AZ1 and AZ2.
  - Don't forget to select the VPC that we created for the project.
  - Select the **Turn on Elastic Load Balancing health checks** checkbox.
  - Select the **Enable group metrics collection within CloudWatch** checkbox.

  ![image](https://github.com/user-attachments/assets/585d6654-77bd-4f2d-89b6-9789e41fe379)

  - As you can see in the image above, the desired capacity is the number of instances that we want to maintain in general, whereas for the scaling limits, this sets the boundaries.
  - For SNS notifications, it's optional, but if you want to add it, simply create a topic first, then you should receive an AWS email to confirm.

  ![image](https://github.com/user-attachments/assets/46bfc62e-ccc1-41b9-8123-f7988691fd8b)

  - After creating the ASG, in the instances page, we will see two EC2 instances that were created. This is because we set the desired capacity to 2.

### Making Sure the Website Works

To ensure the website is functioning correctly, we need to check a few things:

1. EC2 instances are in the target group, and both are healthy.

   ![Screenshot 2024-08-04 073758](https://github.com/user-attachments/assets/e54f7aa4-4f13-4445-a2fd-fff499403a73)

2. You can access the website using the domain name.

   ![Screenshot 2024-08-04 074051](https://github.com/user-attachments/assets/9d37971b-43b1-4993-90a0-de505e68f3b6)

### Making Sure the Auto Scaling Group Works

To test the ASG, we can simply terminate one EC2 instance. Since we set the desired capacity to 2, when we terminate one instance, the auto-scaling group should create another EC2 instance automatically. Let's test that:

- As you can see, it works:

  ![combined_image](https://github.com/user-attachments/assets/0114efe6-d601-4eef-b62a-cd2c5834410e)

---

Currently, the website works perfectly fine, and I will just add a WordPress template to make the website somewhat presentable, although this is not directly related to the project.

![image](https://github.com/user-attachments/assets/75e14db6-9cdd-4aa1-92de-9d944420c50f)

---

## Deleting the Resources

- **ASG:** Delete the ASG first to avoid automatically creating instances. Also, note that deleting the ASG will remove the currently running EC2 instances.

  ![image](https://github.com/user-attachments/assets/e16d9bea-10c4-45f7-a396-5a1cea07b446)

- **ALB (Application Load Balancer):**

  ![image](https://github.com/user-attachments/assets/77ea2c66-4ca2-4d2b-bf70-3326e2221a2e)

- **Target Group:**

  ![image](https://github.com/user-attachments/assets/b5f54cbf-686d-4204-8c6b-e03fc1cd204c)

- **EFS (Elastic File System):**

  ![image](https://github.com/user-attachments/assets/df1f7853-eba5-4233-ae20-f527cc1e4fc8)

- **RDS:** This will take 5-10 minutes to delete.

  ![image](https://github.com/user-attachments/assets/bc2ef922-bd7c-42aa-ae62-965356a87ad9)

- **Subnets Group:**

  ![image](https://github.com/user-attachments/assets/f7a4f9d3-9643-4906-b135-33efb3f26a98)

- **NAT Gateway:**

  ![image](https://github.com/user-attachments/assets/38b5daa6-a5e0-422a-87d8-745a5d14340b)

- **Release the Elastic IP:**

  ![image](https://github.com/user-attachments/assets/81b58f59-7f53-4e8c-bcc4-c04f337365ea)

- **EC2 Instance Endpoints:**

  ![image](https://github.com/user-attachments/assets/efda5296-9f2c-4ab7-8277-191de44441de)

- **Security Groups (DB, EFS, WEB-SERVER, EICE, ALB):**

  ![image](https://github.com/user-attachments/assets/3434bc44-ba33-400f-b004-7a081dd8a40d)

- **VPC:**

  ![image](https://github.com/user-attachments/assets/0b57d99f-1cc5-423e-a46b-e50e3258ab21)

- **IAM User:** I will also delete the IAM user.

  ![image](https://github.com/user-attachments/assets/40778292-4a09-479a-a233-75327f8a0e9d)
