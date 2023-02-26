# Yeasy (Web App)
[**Yeasy**] is a Web-based Application constructed using ***Docker*** through ***AWS*** Platform. [**Yeasy**] Uses EC2 Instances to Host the Application, whereby the Yeasy itself is Running in a Docker Container, placed In-Front of the Container is a Reverse Proxy and Load Balancer done using Nginx.
<hr/><br/>

## Step 1: Creating a S3 Bucket *(3 Steps)*
1) Head to Amazon S3
2) Create a New S3 Bucket with the Targeted Region [**ap-southeast-1** within this Demo] [<S3_BUCKET_REGION_NAME>](#Names)
3) Provide the S3 Bucket a DNS Compliant Name and Create the New Bucket [<S3_BUCKET_NAME>](#Names)
<br/>

## Step 2: Setup IAM Roles for the EC2 Instance *(5 Steps)*
1) Head to IAM
2) Create a New IAM Role
3) Select "AWS Service" for the Trusted Entity, and "EC2" for the Use Case 
4) Select then Add "AmazonSSMMsonS3FullAccess" Policies
5) Provide the Role a Name and Create it [**Role-Yeasy-EC2** within this Demo] [<IAM_ROLE_NAME>](#Names)
<br/>

## Step 3: Launching a New EC2 Instance *(9 Steps)*
1) Head to Amazon EC2
2) Select the Desired Region for the Instance [**ap-southeast-1** within this Demo [<EC2_REGION_NAME>](#Names)
3) Select the Default AMI [**Amazon Linux 2** within this Demo]
4) Select Desired Instance Type [**t2.micro** within this Demo]
5) Accept the Default VPC Network Setting (Ensure to Enable "Auto Assign Public IP"]
6) Select "Proceed without Key Pair" on the Key Pair Section *{Not Connecting through SSH, Proceed without any Key Pair}*
7) Create a New Security Group, Allow Port 80 for HTTP Traffic (Reverse Proxy), Allow Port 8080 also for HTTP Traffic (Load Balancer) and Port 5000 for the Docker Registry
8) Attach the Create IAM Role [**Role-Yeasy-EC2** within this Demo]
9) Place the Following Script in the "User Data" Section and Create the Instance
```
#!/bin/bash

sudo yum update -y
sudo amazon-linux-extras install docker nginx1 -y

# Create Directory for Docker Compose Files
sudo mkdir -p /opt/docker/registry
sudo touch /opt/docker/registry/docker-compose.yml /opt/docker/registry/.env /etc/nginx/conf.d/yeasy-web.conf

# Install Docker-Compose
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Start Docker and Nginx Services
sudo systemctl enable docker nginx
sudo systemctl start docker nginx
```
<br/>

## Step 4: Configuring the EC2 Instance for Docker Registry *(10 Steps)*
1) Wait for the EC2 Instance to Start and Confirm that both Status Check have Passed
2) Select the EC2 Instance then Connect Using the "Connect" Button
3) Connect either using "EC2 Instance Connect" OR "Session Manager" [**Session Manager** within this Demo]
4) Change the User to the "ec2-user" once Connected
```
sudo su - ec2-user
```
5) Edit the **docker-compose** File Create during EC2 Setup
```
sudo nano /opt/docker/registry/docker-compose.yml
```
6) Paste in the Following Script within the Nano Text Editor then Save the **docker-compose.yml** File
```
version: '3'

services:
  yeasy:
    image: registry:2
    ports:
      - 5000:5000
    container_name: yeasy
    restart: always
    env_file:
      - ./.env
```
7) Edit the **.env** File which Consists the Environment Variable required by Docker
```
sudo nano /opt/docker/registry/.env
```
8) Paste in the Following Script then Save the **.env** File, Follow the Created <a name="Names"></a> Names
```
REGISTRY_STORAGE=s3
REGISTRY_STORAGE_S3_REGION=<S3_BUCKET_REGION_NAME>
REGISTRY_STORAGE_S3_BUCKET=<S3_BUCKET_NAME>
AWS_REGION=<EC2_REGION_NAME>
AWS_CONTAINER_CREDENTIALS_RELATIVE_URI=/v2/credentials/<IAM_ROLE_NAME>
``` 
9) Execute the **"docker-compose"" command and Run the Registry in the Background
```
cd /opt/docker/registry/ && sudo /usr/local/bin/docker-compose up -d
```
10) Verify that the Container is Running through the Command (Container should have the Status of Up)
```
sudo docker ps
```
<br/>

## Step 5: Pulling the Yeasy Webapp Container and Uploading to the Private Registry *(3 Steps)*
1) Pull the yeasy/webapp Container from Dockerhub
```
sudo docker pull yeasy/simple-web
```
2) Tag the Yeasy Webapp Container with the Private Docker Registry that is Running, Then Push the Container to the Private Repository
```
sudo docker tag yeasy/simple-web localhost:5000/yeasy/simple-web
sudo docker push localhost:5000/yeasy/simple-web
```
3) Verify that the Yeasy Webapp is Pushed Successfully to the Private Docker Registry in the Create S3 Bucket
```
curl -kX GET http://localhost:5000/v2/_catalog
```
<br/>

## Step 6: Setup Nginx as a Reserve Proxy and Load Balancer to the WebApp *(6 Steps)*
1) Run the 3 Yeasy Webapp Containers
```
sudo docker run -d -p 7777:80 --name yeasy-webapp-1 localhost:5000/yeasy/simple-web
sudo docker run -d -p 8888:80 --name yeasy-webapp-2 localhost:5000/yeasy/simple-web
sudo docker run -d -p 9999:80 --name yeasy-webapp-3 localhost:5000/yeasy/simple-web
```
2) Access the Nginx Config File
```
sudo nano /etc/nginx/conf.d/yeasy-web.conf
```
3) Run the Following Script
```
# Configuration of the Reverse Proxy Server List
upstream yeasy-reverse-proxy {
    server 0.0.0.0:8888;
}

server {
    listen 80;
    server_name $host;

    location / {
        proxy_pass http://yeasy-reverse-proxy;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}



# Load balancer Server List configuration 
upstream yeasy-lb {
    least_conn;
    server 0.0.0.0:7777;
    server 0.0.0.0:9999;
    # Add more servers as needed
}


server {
    listen 8080;
    server_name $host;

    location / {
        proxy_pass http://yeasy-lb;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
4) Reload the Nginx Service
```
sudo systemctl restart nginx
```
5) Access the Public IP of the EC2 Instance and Browse the Webpage (Through HTTP)
6) Browsing the Webpage using **Port 80** will go Through the Reverse Proxy; While Browsing using **Port 8080** Goes through the Load Balancer
```
http://<EC2_Public_IP>:80
http://<EC2_Public_IP>:8080
```
<br/>

