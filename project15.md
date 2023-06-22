
## Networking
- Create VPC
- Create Internet Gateway
- Create 8 Subnets
  | Network Layer |  AZ A | AZ B |
  | --- | --- | --- |
  | Public Layer | Subnet 1 | Subnet 2 |
  | Private - Proxy Layer | Subnet 3 | Subnet 4 |
  | Private - Compute Layer | Subnet 5 | Subnet 6 |
  | Private - Data Layer | Subnet 7 | Subnet 8 | 
- Create a Route Table. Edit Routes. For Destination allow any ip `0.0.0.0/0`. Under Target, select the Internet Gateway you created earlier. Internet Gateway is two-way, i.e. allows the subnet to connect to the internet and for the internet to connect to the subnet. Associate Public Layer Subnets to this Route Table.
- Create a NAT Gateway. Allocate new Elastic Public IP address.
- Create another Route Table. Edit routes. For Destination allow any ip `0.0.0.0/0`. Under Target, select the NAT Gateway you created earlier.Associate `Proxy Layer` and `Compute Layer` Private Subnets to this Route Table to allow subnets communicate with the NAT gateway. NAT Gateway is one-way. Only allows the subnet to the internet.


## Firewall
- Create security group for Bastion server. Open port `22` for `SSH` and limit connections to be from `my IP`
- Create security group for the Public ALB Load Balancer. Open ports `80` for `http` and `443` for `https` connections from anywhere `0.0.0.0/0`
- Create security group for Nginx Reverse Proxy server. Open port `22` for `SSH` and limit connections to be from Bastion Server only. Open ports `80` for `http` and `443` for `https` traffic from External Load Balancer Server only.
- Create security group for the Internal Load Balancer. Open ports `80` for `http` and `443` for `https` connections from Nginx Reverse Proxy Servers Only.
- Create security groups for the Websites. One Sg for Wordpress. Another for Tooling. Open ports `80` for `http` and `443` for `https` connections from Internal Load Balancer Servers Only. Open port `22` for `SSH` and limit connections to be from Bastion Server only.
- Create security group for the data layer. Open ports `3306` for `MYSQL` connections from Webservers. 
- Create security group for EFS. Open port `2049` for `EFS` mounting from webservers.



## Compute Resources
### Create & Configure Public & Internal Application Load Balancers, Target Groups, & Auto-Scaling Groups
- In EC2 create Application Load Balancer
- - Select the right VPC
- - Select Availabilty Zones and public Subnets in Mappings
- - Select security group created for Public ALB previously 
- - Add listener for `HTTP` and `HTTPS`. On each, create a target group for the Nginx proxies
- Update VPC settings to enable DNS Hostnames & Update public subnet settings to auto assign public IP address
- Create Key Pair for SSH
- Create a Launch Template for Bastion (Use a redhat based AMI) (include subnet settings for auto scaling in templates) Instance type - Micro
- Create ASG for Bastion
- Create a Launch Template for nginx (Use a Ubuntu based AMI) (Private network)
- Create ASG for nginx
- Connect to server and Install nginx
- Configure subdomain in Route53 to resolve to Public ALB CNAME (which forwards to the target group that has the Nginx Proxy servers)
- Visit subdomain in browser. It should rforward you to the default Nginx server page. Check nginx access logs:
  ```
  tail -f /var/log/nginx/access.log
  ```

- Create default target group for the internal ALB
- Create target group for Wordpress site
- Create target group for tooling site
- Create internal ALB and configure listeners with Host header rules for default web page, wordpress and tooling sites


### Configure Proxy

sudo apt update -y 
sudo apt install nginx -y
sudo unlink /etc/nginx/sites-enabled/default

### Configure Tooling Reverse Proxy

```
sudo vi /etc/nginx/sites-available/tooling-reverse-proxy.conf
```

```
server {
    listen 80;
    server_name tooling.partsunltd.soladaniel.com;
    location / {
        proxy_pass http://internal-punltd-internal-alb-685834512.us-east-1.elb.amazonaws.com/;
        proxy_set_header Host $host;
    }
  }
```

```
sudo ln -s /etc/nginx/sites-available/tooling-reverse-proxy.conf /etc/nginx/sites-enabled/tooling-reverse-proxy.conf
```

### Configure WordPress Site Reverse Proxy

```
sudo vi /etc/nginx/sites-available/wpsite-reverse-proxy.conf
```

```
server {
    listen 80;
    server_name wpsite.partsunltd.soladaniel.com;
    location / {
        proxy_pass http://internal-punltd-internal-alb-685834512.us-east-1.elb.amazonaws.com/;
        proxy_set_header Host $host;
    }
  }
```

```
sudo ln -s /etc/nginx/sites-available/wpsite-reverse-proxy.conf /etc/nginx/sites-enabled/wpsite-reverse-proxy.conf
```


- Create Launch Template for Tooling ASG (Ensure the IAM for instance profile is configured) (RedHat Linux)
- Create Launch Template for Wordpress ASG (Ensure the IAM for instance profile is configured)  (RedHat Linux)
- Create ASG for Tooling instances
- Create ASG for Wordpress instances
- Configure instance profile and give the tooling and wordpress instances relevant permissions to access AWS resources (for example, S3, EFS)
- Configure nginx to upstream to internal ALB


## Configure & Mount EFS
- Create an Elastic File System. In the Network settings, select `Private Subnet 1` and `Private Subnet 2` (the subnets where the webservers will exist - This is compulsory with EFS) for the Mount Targets. Attached the Data Layer security group.
- Create two Access Points on the EFS for the two apps we intend to deploy on this infrastructure. One Access Point for the Wordpress website, the second for the Tooling Wesite. This is to ensure that the file structure is clean and files from both apps do not overwrite themselves.

## Configure RDS
- Create a Certificate for the domain to be attached to the Application Load Balancers later on.
- To create an RDS instance there are two prerequisites. KMS Key and Subnet groups
- - Create a KMS key from Key Management Service (KMS) to be used to encrypt the database instance. Use the `Symmetric` key type. Set Key Administrative and Usage Permissions to role or user
- - Navigate to AWS RDS and create a `DB Subnet Group`. Select the `VPC`, `Availability Zones`, and the `Private Subnets` where the databases are to be hosted. In this case, `Private Subnet 3` and `Private Subnet 4`
- Create a database. Be sure to choose the right `VPC`, `Subnet Group`, and `Security Group`

