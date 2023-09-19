## PREREQUISITES

### Before starting off, you need to properly configure your AWS account and Organization unit using the steps below:

### Step 1:
```
- Create an AWS Master account. (Also known as Root Account)
- Within the Root account, create a sub-account and name it DevOps. (You will need another email address to complete this)
- Within the Root account, create an AWS Organization Unit (OU). Name it Dev. (We will launch Dev resources in there)
- Move the DevOps account into the Dev OU.
- Login to the newly created AWS account using the new email address.
```

### Step 2:
- Get a domain for your fictitious company

- Create a hosted zone in AWS, and map it to your free domain
![](./project15_images/hosted%20zone.jpg)

![project 15 overview](./project15_images/project15_overview.png)

### The diagram above is what will be implemented in this project.
- A client from the internet visits your domain name i.e mannzeus.shop, the traffic will go to route 53 which resolves the name to the DNS NAME of your ALB and route the traffic to ALB via the internet gateway of your VPC. i.e Route 53 resolves mannzeus.shop to the DNS name of your ALB and sends the traffic to your ALB

- The internet-facing ALB receives the traffic and forwards it to the Reverse-Proxy servers. The reverse-proxy servers will be configured to forward all received traffic to the internal ALB.

- Since you have 2 webservers serving 2 different websites behind the internal ALB, you will configure a rule for your internal ALB to check for host headers to enable it forward the match the right traffic to the right webserver. 
- The Host header will be configured in the Nginx Reverse-Proxy servers. You will set 2 rules , 1 will be the default rule saying "Forward all traffic to mannzeus.shop target". The 2nd Rule will be a condition saying "check the host headers, if it is tooling, forward traffic to the tooling server(target).
- Your webservers will be using the EFS to serve their content and storing data in database managed by RDS.

### Create a VPC, edit VPC settings and enable DNS hostnames.
![](./project15_images/edit%20VPC%20settings.jpg)
![](./project15_images/edit%20VPC%20settings2.jpg)

### Create an internet gateway and attach to your VPC
![](./project15_images/create%20and%20attach%20igw.jpg)

### Create subnets, from your project diagram you are to create 2 public subnets and 4 private subnets IN 2 Availability Zones
```
10.0.1.0/24 public subnet 1 AZ A
10.0.2.0/24 public subnet 2 AZ B

10.0.3.0/24 private subnet 1 AZ A
10.0.4.0/24 private subnet 2 AZ B
10.0.5.0/24 private subnet 3 AZ A
10.0.6.0/24 private subnet 4 AZ B
```
![](./project15_images/create%20subnets.jpg)

### Create 2 route tables, a public and private route table and associate them to the public and private subnets respectively. Proceed to edit the routes, Edit a route in public route table, and associate it with the Internet Gateway. (This is what allows a public subnet to be accisble from the Internet)

![edit public route](./project15_images/edit%20public%20route.jpg)
### Create a NAT gateway in any of your public subnets and attach an elastic IP to it
### Create 2 bastion hosts, one in each AZ and attach elastic IPs to both

### Next you create security groups for the resources in your infrastructure 
- Security group for your external ALB: This accepts traffic from the whole internet so your inbound rule has to accept HTTPS and HTTP traffic from the anywhere.

![ext alb secu](./project15_images/ext%20alb%20secu.jpg)

- Security Group for the bastion server: Access to the Bastion servers should be allowed only from workstations that need to SSH into the bastion servers. Hence, you can use your workstation public IP address. To get this information, simply go to your terminal and type `curl www.canhazip.com`
![jump server secu](./project15_images/jump%20server%20secu.jpg)

- Security Group for the reverse proxy server: The reverse proxy only accepts traffic from the external ALB, thus it's inbound rule should allow HTTP AND HTTPS traffic only from the external ALB security group. You will need to make some configurations on the reverse proxy server thus, an inbound rule that allows SSH access from your bastion server to reverse proxy server should also be made
![reverse proxy](./project15_images/reverse%20proxy.jpg)

- Security Group for the internal ALB: This too accepts HTTP and HTTPS traffic only from the reverse proxy server  , create the rules accordingly
![int alb](./project15_images/internal%20ALB.jpg)

- Security Group for the webservers: Both webservers are in private subnets and accepts HTTP and HTTPS traffic only from the internal ALB and SSH traffic from your bastion. Thus create the rules accordingly

- Security Group for the Data layer: This layer is comprised of Amazon Relational Database Service (RDS) and Amazon Elastic File System (EFS) and must be carefully designed. The webservers should be able to connect to RDS and the EFS Mountpoint. The Bastion server also needs MySQL/Aurora administrative access to the database server.
A rule for MySQL/aurora access for the bastion is made
A rule for NFS access for the webservers is made
A rule for MySQL/aurora for the webservers is made
![](./project15_images/datalayer.jpg)

### Now that the seurity groups are created, get to creating the resources in your Infrastructure, create a certificate, an EFS and RDS.

### Go create a certificate
![](./project15_images/certificate.jpg)
 
 ### Create your filesystem, set the mount targets to the specific subnets where your webservers will be located, in this case private subnet 1 and 2
 ![create efs1](./project15_images/create%20efs1.jpg)
 ![create efs2](./project15_images/create%20efs2.jpg)

 ### After creating the file systems, create access points, this is what the webservers mounts to the EFS with. Create 2 access points as you have 2 different websites
 ![FS accesspoint](/project15_images/FS%20access%20point.jpg)

 ### Next Create your RDS, before creating your RDS you first need to create a KMS key from the KMS store and a subnet rule.

 ### First create a key to be used to encrypt the RDS
 ![create key](./project15_images/create%20key.jpg)

 ### Create a DB subnet group, choose the private subnets where the RDS is located in your infrastruture  
 ![subnet groups](./project15_images/subnet%20groups.jpg)

 ### Create the database with MySQL engine

 ### Next you need to create an ami to be used in creating the launch template. create 3 redhat instances: a webserver, a bastion and nginx
 ![ami's](./project15_images/amis.jpg)
 
 ## PREPARE THE BASTION SERVER AMI

 ### Bastion ami installation
-------------------------------------
 ```
 sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

yum install -y https://dl.fedoraproject.org/pub/epel/epel-next-release-latest-9.noarch.rpm

yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm

yum install wget vim python3 telnet htop git mysql net-tools chrony -y

systemctl start chronyd
systemctl enable chronyd
 ```

## PREPARE NGINX SERVER AMI

# Nginx ami installation 
-----------------------------------------

 sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

yum install -y https://dl.fedoraproject.org/pub/epel/epel-next-release-latest-9.noarch.rpm

yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm

yum install wget vim python3 telnet htop git mysql net-tools chrony -y

systemctl start chronyd
systemctl enable chronyd

### configure selinux policies for the webservers and nginx servers to function properly
```
setsebool -P httpd_can_network_connect=1
setsebool -P httpd_can_network_connect_db=1
setsebool -P httpd_execmem=1
setsebool -P httpd_use_nfs 1
```

### Install amazon efs utils for mounting the target on the Elastic file system
```
git clone https://github.com/aws/efs-utils

cd efs-utils

yum install -y make

yum install -y rpm-build

make rpm 

yum install -y  ./build/amazon-efs-utils*rpm
```
### set up a self-signed certificate for the nginx instance
```
sudo mkdir /etc/ssl/private

sudo chmod 700 /etc/ssl/private

openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/ACS.key -out /etc/ssl/certs/ACS.crt

sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```
![create certificate](./project15_images/create%20certificate.jpg)
![create certificate](./project15_images/create%20certificate2.jpg)
![create certificate3](./project15_images/create%20certificate3.jpg)

## PREPARE THE WEBSERVER AMI

sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm

yum install -y https://dl.fedoraproject.org/pub/epel/epel-next-release-latest-9.noarch.rpm

yum install -y dnf-utils http://rpms.remirepo.net/enterprise/remi-release-9.rpm

yum install wget vim python3 telnet htop git mysql net-tools chrony -y

systemctl start chronyd
systemctl enable chronyd

### configure selinux policies for the webservers and nginx servers to function properly
```
setsebool -P httpd_can_network_connect=1
setsebool -P httpd_can_network_connect_db=1
setsebool -P httpd_execmem=1
setsebool -P httpd_use_nfs 1
```
### The webserver will host the websites using apache, so set up self-sogned certificate for the apache  webserver instance
```
yum install -y mod_ssl

openssl req -newkey rsa:2048 -nodes -keyout /etc/pki/tls/private/ACS.key -x509 -days 365 -out /etc/pki/tls/certs/ACS.crt

```
### For Nginx you specify the configuration in the `reverse.conf` file, but for apache, you edit the ssl/conf file. Look for the section where you specify the certificate path and the ssl key path
`vi /etc/httpd/conf.d/ssl.conf`
![EDIT SSL CONF](./project15_images/EDIT%20SSL%20CONF.jpg)

### Create images from the 3 instances
![ami's created](./project15_images/ami's%20created.jpg)

## Now create Target groups for the ALB. From the project diagram you can see that we have 3 sets of Target groups, the Nginx Target Group, the Wordpress server target groups and the Tooling server Target Groups. Thus you will be creating 2 target groups
- Select Instances as the target type
- Ensure the protocol HTTPS on secure TLS port 443
- Ensure that the health check path is /healthstatus
- Register Nginx Instances as targets
- Ensure that health check passes for the target group

![target groups](./project15_images/target%20groups.jpg)

### Create the Application Load Balancers next, both the External and Internal
When creating the Internal ALB you will set to set a rule to check for host headers and forward traffic to the appropriate target group.


https://github.com/NyerhovwoOnitcha/ACS/assets/101157174/148b278e-56cb-41ca-a4e5-72c4e8ebecd7




https://github.com/NyerhovwoOnitcha/ACS/assets/101157174/765648f3-006a-4471-834d-120c8b4529b2


### Next Create the launch Template and autoscaling group



















