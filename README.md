# AWS-Three-Tier-Web-Architecture

Architecture Overview
<img width="1093" height="493" alt="image" src="https://github.com/user-attachments/assets/988d0efc-e9a9-4cb6-99aa-195ce05cb7c7" />

In this architecture, a public-facing Application Load Balancer forwards client traffic to our web tier EC2 instances.
The web tier is running Nginx webservers that are configured to serve a React.js website and redirects our API calls
to the application tier’s internal facing load balancer. The internal facing load balancer then forwards that traffic
to the application tier, which is written in Node.js. The application tier manipulates data in an Aurora MySQL multi-AZ
database and returns it to our web tier. Load balancing, health checks and autoscaling groups are created at each layer
to maintain the availability of this architecture.

1. VPC Creation
<img width="625" height="318" alt="image" src="https://github.com/user-attachments/assets/54fac42e-443f-4861-8b58-0cab77d9253b" />

2. Create Subnet.
  We will need six subnets across two availability zones.
  Public-Web-Subnet-AZ-1, Private-App-Subnet-AZ-1, Private-DB-Subnet-AZ-1, Public-Web-Subnet-AZ-2, Private-App-Subnet-AZ-2,
  Private-DB-Subnet-AZ-2.

<img width="625" height="173" alt="image" src="https://github.com/user-attachments/assets/1f485b72-0a41-43e7-88df-bbd621622a20" />

3. Create Internet Gateway and Attach to VPC.
  <img width="638" height="93" alt="image" src="https://github.com/user-attachments/assets/22abd41a-b813-4096-88a6-5396255aa898" />

4. Create NAT Gateway.
   In order for our instances in the app layer private subnet to be able to access the internet they will
   need to go through a NAT Gateway. For high availability, you’ll deploy one NAT gateway in each of your
   public subnets.
  <img width="634" height="115" alt="image" src="https://github.com/user-attachments/assets/07efcc97-ef90-4a08-a638-6399bdbb273a" />

5. Create route table.
  let’s create one route table for the web layer public subnets.
  <img width="636" height="96" alt="image" src="https://github.com/user-attachments/assets/7dbd2827-60d5-4c72-a04b-bc4d1860b4c1" />

  Add a route that directs traffic from the VPC to the internet gateway.
  <img width="621" height="120" alt="image" src="https://github.com/user-attachments/assets/ae182608-d269-46df-a942-fd9052024cd6" />
  
  Select Subnet Associations and click Edit subnet associations. Select the two web layer public subnets you created eariler and click Save associations.

  <img width="637" height="126" alt="image" src="https://github.com/user-attachments/assets/0b592385-eee3-4c2a-8044-a88bcb0fa811" />
  
  Now create 2 more route tables, one for each app layer private subnet in each availability zone. These route tables will route app layer traffic destined for outside the VPC to the NAT gateway in the respective availability zone
  <img width="637" height="352" alt="image" src="https://github.com/user-attachments/assets/996eaef6-6473-42ca-9f87-8f30da5768c4" />

  <img width="639" height="232" alt="image" src="https://github.com/user-attachments/assets/44c3a65a-fb18-486c-91e9-080ec2f28bce" />

7. Security Groups.
  The first security group you’ll create is for the public, internet facing load balancer. After typing a name and description, add an inbound rule to allow HTTP type traffic for your IP.
  <img width="708" height="212" alt="image" src="https://github.com/user-attachments/assets/343cd2e2-f618-4848-8c80-f837ae06e5c6" />

  The second security group you’ll create is for the public instances in the web tier. After typing a name and description, add an inbound rule that allows HTTP type traffic from your internet facing load balancer security group you created in the previous step. This will allow traffic from your public facing load balancer to hit your instances. Then, add an additional rule that will allow HTTP type traffic for your IP. This will allow you to access your instance when we test.
  <img width="708" height="229" alt="image" src="https://github.com/user-attachments/assets/661d4e66-05d3-4597-982c-918a4a078c21" />

  The third security group will be for our internal load balancer. Create this new security group and add an inbound rule that allows HTTP type traffic from your public instance security group. This will allow traffic from your web tier instances to hit your internal load balancer.
  
  <img width="691" height="214" alt="image" src="https://github.com/user-attachments/assets/c6af0628-8d64-4410-9b34-0c6e718641cb" />

  The fourth security group we’ll configure is for our private instances. After typing a name and description, add an inbound rule that will allow TCP type traffic on port 4000 from the internal load balancer security group you created in the previous step. This is the port our app tier application is running on and allows our internal load balancer to forward traffic on this port to our private instances. You should also add another route for port 4000 that allows your IP for testing.
  <img width="696" height="207" alt="image" src="https://github.com/user-attachments/assets/2a288359-c92f-438e-92eb-18f0f20be8d8" />

  The fifth security group we’ll configure protects our private database instances. For this security group, add an inbound rule that will allow traffic from the private instance security group to the MYSQL/Aurora port (3306).
  <img width="708" height="214" alt="image" src="https://github.com/user-attachments/assets/19169521-9bb0-4a76-b936-b221574d082b" />

8. Database Deployment.
  Navigate to the RDS dashboard in the AWS console and click on Subnet groups on the left hand side. Click Create DB subnet group.
  <img width="652" height="371" alt="image" src="https://github.com/user-attachments/assets/37daf4a0-ae99-4111-be1c-4700c0d7c3d4" />

  Create database.
  <img width="1310" height="512" alt="image" src="https://github.com/user-attachments/assets/80394dd2-a920-4ef2-a868-3596bc7fc591" />
  <img width="690" height="370" alt="image" src="https://github.com/user-attachments/assets/c1858548-113c-43f6-a3dd-c5676c71f1f0" />
  
  When your database is provisioned, you should see a reader and writer instance in the database subnets of each availability zone. Note down the writer endpoint for your database for later use.
  <img width="839" height="206" alt="image" src="https://github.com/user-attachments/assets/4480e530-fce6-49b9-bad6-f28ed1ecb9b6" />

9. App Instance Deployment.

  IAM EC2 Instance Role Creation. Adding permissions.
  AmazonSSMManagedInstanceCore,
  AmazonS3ReadOnlyAccess
  <img width="637" height="473" alt="image" src="https://github.com/user-attachments/assets/1b58f51d-804c-470d-8925-8db6220ba657" />

  Select Amazon Linux for the OS and Amazon Linux 2 AMI, elect t2.micro. (Select Proceed without a key pair.)
  <img width="673" height="252" alt="image" src="https://github.com/user-attachments/assets/72ca2cde-11ed-4522-a8d4-c9ee560d5e64" />

  For Network settings, click Edit. Select a VPC and a subnet. For Subnet, select Private-App-Subnet. For Security Group, select PrivateInstanceSG, which you created in the previous step. For Auto-assign public IP, select Disable. click Advanced details.
  <img width="665" height="354" alt="image" src="https://github.com/user-attachments/assets/4142f874-c2d7-418e-b9ef-a7297c4c6723" />
  <img width="682" height="215" alt="image" src="https://github.com/user-attachments/assets/1f5c873b-b663-4c1b-b594-c5f2de8139cf" />

Connect to Instance.

<img width="622" height="156" alt="image" src="https://github.com/user-attachments/assets/925088c2-fe8d-40fc-a8b0-ea10cb695132" />

  When you first connect to your instance like this, you will be logged in as ssm-user which is the default user. Switch to ec2-user by executing the following command in the browser terminal:
  
  sudo -su ec2-user
  
  If your network is configured correctly up till this point, you should be able to ping the google DNS servers.
  
  ping 8.8.8.8
  
  <img width="474" height="139" alt="image" src="https://github.com/user-attachments/assets/3287811f-b3cd-48c6-94ba-9cfe21c82b79" />

10. Configure Database.
  Start by downloading the MySQL CLI:
```
  sudo yum install mariadb105
  ```
  <img width="627" height="199" alt="image" src="https://github.com/user-attachments/assets/1107a123-e1a6-4eb3-8dca-ace358186156" />

  Connect to mysql.
```
  mysql -h CHANGE-TO-YOUR-RDS-ENDPOINT -u CHANGE-TO-USER-NAME -p
```
  <img width="626" height="63" alt="image" src="https://github.com/user-attachments/assets/4e72769d-12bf-40c5-8095-49948b54d0df" />
  
```
  CREATE DATABASE webappdb;
  
  SHOW DATABASES;
  
  USE webappdb;    
```
  Then, create the following transactions table by executing this create table command:
```
  CREATE TABLE IF NOT EXISTS transactions(id INT NOT NULL
  AUTO_INCREMENT, amount DECIMAL(10,2), description
  VARCHAR(100), PRIMARY KEY(id));    
```
  Verify the table was created:
```
  SHOW TABLES;    
```
  Insert data into table for use/testing later:
```
  INSERT INTO transactions (amount,description) VALUES ('400','groceries');   
```
  <img width="625" height="258" alt="image" src="https://github.com/user-attachments/assets/3478c50b-e027-4442-b1b1-6b90a4d08f45" />


  Verify that your data was added by executing the following command.
  ```
  SELECT * FROM transactions;
```
  When finished, just type exit and hit enter to exit the MySQL client.

11. Configure App Instance.

  Download the code.
```
  git clone https://github.com/aws-samples/aws-three-tier-web-architecture-workshop.git
```
  <img width="625" height="92" alt="image" src="https://github.com/user-attachments/assets/1711a9e6-d374-41b7-8077-633f71c5b4c7" />
  
  create a new S3 bucket.
  
  <img width="646" height="146" alt="image" src="https://github.com/user-attachments/assets/50b99d81-90da-45f8-9311-003de4110045" />

  The first thing we will do is update our database credentials for the app tier. To do this, open the application-code/app-tier/DbConfig.js file from the github repo in your favorite text editor on your computer. You’ll see empty strings for the hostname, user, password and database. Fill this in with the credentials you configured for your database, the writer endpoint of your database as the hostname, and webappdb for the database. Save the file.

  <img width="639" height="130" alt="image" src="https://github.com/user-attachments/assets/2a1acf7a-8c25-40b8-9134-f22c5aa531ac" />

  Upload the app-tier folder to the S3 bucket
  
  <img width="625" height="110" alt="image" src="https://github.com/user-attachments/assets/cfac5606-b490-41cb-bc31-acbf9430c12e" />

  Go back to your SSM session. Now we need to install all of the necessary components to run our backend application. Start by installing NVM (node version manager).

  ```
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
  source ~/.bashrc
  ```

  Next, install a compatible version of Node.js and make sure it's being used
```
  nvm install 16
nvm use 16
```
PM2 is a daemon process manager that will keep our node.js app running when we exit the instance or if it is rebooted. Install that as well.
```
npm install -g pm2   
```
Now we need to download our code from our s3 buckets onto our instance. In the command below, replace BUCKET_NAME with the name of the bucket you uploaded the app-tier folder to:
```
cd ~/
aws s3 cp s3://BUCKET_NAME/app-tier/ app-tier --recursive
```
<img width="620" height="153" alt="image" src="https://github.com/user-attachments/assets/94bca3b4-e1f5-4a7d-93b2-9fb12d524ec8" />


Navigate to the app directory, install dependencies, and start the app with pm2.
```
cd ~/app-tier
npm install
pm2 start index.js
```
To make sure the app is running correctly run the following:
```
pm2 list
```
If you see a status of online, the app is running. If you see errored, then you need to do some troubleshooting. To look at the latest errors, use this command:
```
pm2 logs
```
NOTE: If you’re having issues, check your configuration file for any typos, and double check that you have followed all installation commands till now.

Right now, pm2 is just making sure our app stays running when we leave the SSM session. However, if the server is interrupted for some reason, we still want the app to start and keep running. This is also important for the AMI we will create:
```
pm2 startup
```
After running this you will see a message similar to this.

[PM2] To setup the Startup Script, copy/paste the following command: sudo env PATH=$PATH:/home/ec2-user/.nvm/versions/node/v16.0.0/bin /home/ec2-user/.nvm/versions/node/v16.0.0/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user —hp /home/ec2-user

DO NOT run the above command, rather you should copy and past the command in the output you see in your own terminal. After you run it, save the current list of node processes with the following command:
```
pm2 save
```

12. Test App Tier

Now let's run a couple tests to see if our app is configured correctly and can retrieve data from the database.

To hit out health check endpoint, copy this command into your SSM terminal. This is our simple health check endpoint that tells us if the app is simply running.
```
curl http://localhost:4000/health
```
The response should looks like the following:

"This is the health check"

Next, test your database connection. You can do that by hitting the following endpoint locally:
```
curl http://localhost:4000/transaction
```
You should see a response containing the test data we added earlier:

{"result":[{"id":1,"amount":400,"description":"groceries"},{"id":2,"amount":100,"description":"class"},{"id":3,"amount":200,"description":"other groceries"},{"id":4,"amount":10,"description":"brownies"}]}

<img width="619" height="66" alt="image" src="https://github.com/user-attachments/assets/aff532e1-3ba6-4192-8292-0218162ab184" />

If you see both of these responses, then your networking, security, database and app configurations are correct.

Your app layer is fully configured and ready to go.

13. App Tier AMI

Select the app tier instance we created and under Actions select Image and templates. Click Create Image.
  
<img width="627" height="216" alt="image" src="https://github.com/user-attachments/assets/bb5d6278-6cf2-462e-bc98-59cf03c725e7" />

14. Create Target group

<img width="632" height="504" alt="image" src="https://github.com/user-attachments/assets/2d3d86db-8e76-4987-9b84-e2bea857b2bd" />

set the protocol to HTTP and the port to 4000. Remember that this is the port our Node.ja app is running on. Select the VPC we've been using thus far, and then change the health check path to be /health. This is the health check endpoint of our app. Click Next.

<img width="626" height="229" alt="image" src="https://github.com/user-attachments/assets/a093402e-6e65-48f0-ba33-abbe5ac39b68" />

We are NOT going to register any targets for now, so just skip that step and create the target group.

15. Create Internal Load Balancer.

<img width="626" height="301" alt="image" src="https://github.com/user-attachments/assets/763dd140-22b5-4836-8171-c43eb1bc1f96" />

Select the correct network configuration for VPC and private subnets.

<img width="618" height="251" alt="image" src="https://github.com/user-attachments/assets/29698c29-c63b-4006-bb96-39f2defe676d" />

Select the security group we created for this internal ALB. Now, this ALB will be listening for HTTP traffic on port 80. It will be forwarding the traffic to our target group that we just created, so select it from the dropdown, and create the load balancer.

<img width="644" height="247" alt="image" src="https://github.com/user-attachments/assets/a78946b0-58f3-4dd5-bb28-611dea854e01" />

<img width="644" height="242" alt="image" src="https://github.com/user-attachments/assets/c2dcb4bb-cdcf-4ed0-b5d0-422a9466aca7" />

16. Create Launch Template.

Under Application and OS Images include the app tier AMI you created.

<img width="543" height="463" alt="image" src="https://github.com/user-attachments/assets/445ca6e9-0e94-45aa-a9a8-fe8372034b44" />

Under Instance Type select t2.micro. For Key pair and Network Settings don't include it in the template. We don't need a key pair to access our instances and we'll be setting the network information in the autoscaling group.

<img width="560" height="398" alt="image" src="https://github.com/user-attachments/assets/78137af9-75ca-407d-a5e1-8feceec3a9d5" />

17. Create Auto Scaling.

<img width="559" height="204" alt="image" src="https://github.com/user-attachments/assets/004a8043-63e9-41ca-a7a0-bbf970dfffed" />

<img width="513" height="464" alt="image" src="https://github.com/user-attachments/assets/a31001f2-b991-4d1f-b29d-1154b4d29f9f" />

<img width="511" height="284" alt="image" src="https://github.com/user-attachments/assets/0d9abca2-e4cc-4eac-83f9-802b968cbc1d" />

You should now have your internal load balancer and autoscaling group configured correctly. You should see the autoscaling group spinning up 2 new app tier instances. If you wanted to test if this is working correctly, you can delete one of your new instances manually and wait to see if a new instance is booted up to replace it.

18. Update Config File.

Before we create and configure the web instances, open up the application-code/nginx.conf file from the repo we downloaded. Scroll down to line 58 and replace [INTERNAL-LOADBALANCER-DNS] with your internal load balancer’s DNS entry. You can find this by navigating to your internal load balancer's details page.

<img width="365" height="197" alt="image" src="https://github.com/user-attachments/assets/1f12e5ad-afa3-4528-8b13-52dfbc42681f" />

<img width="474" height="152" alt="image" src="https://github.com/user-attachments/assets/b12f912f-8826-4f82-9d16-b83f49147b57" />

upload  web-tier and nginx.conf to s3.

<img width="553" height="157" alt="image" src="https://github.com/user-attachments/assets/1f0bcedd-98fd-45a7-b6fb-4dc1731613aa" />

19. Web Instance Deployment.

<img width="625" height="333" alt="image" src="https://github.com/user-attachments/assets/5e39d8b6-a9c5-4a7d-9c0d-053363ea0c72" />

<img width="508" height="451" alt="image" src="https://github.com/user-attachments/assets/354ae3b3-e3bf-47cc-888c-f753b3b9f2c8" />

Connect to Instance.

<img width="592" height="296" alt="image" src="https://github.com/user-attachments/assets/1a62f954-59ce-437a-b50f-f101263e5855" />

Configure Web Instance.

We now need to install all of the necessary components needed to run our front-end application. Again, start by installing NVM and node :
```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
source ~/.bashrc
nvm install 16
nvm use 16
```
Now we need to download our web tier code from our s3 bucket:
```
cd ~/
aws s3 cp s3://BUCKET_NAME/web-tier/ web-tier --recursive
```
<img width="561" height="75" alt="image" src="https://github.com/user-attachments/assets/04a4c410-9aba-4503-87f3-20f8b915a238" />

Navigate to the web-layer folder and create the build folder for the react app so we can serve our code:
```
cd ~/web-tier
npm install 
npm run build
```
NGINX can be used for different use cases like load balancing, content caching etc, but we will be using it as a web server that we will configure to serve our application on port 80, as well as help direct our API calls to the internal load balancer.
```
sudo amazon-linux-extras install nginx1 -y
```
We will now have to configure NGINX. Navigate to the Nginx configuration file with the following commands and list the files in the directory:
```
cd /etc/nginx
ls
```
You should see an nginx.conf file. We’re going to delete this file and use the one we uploaded to s3. Replace the bucket name in the command below with the one you created for this workshop:
```
sudo rm nginx.conf
sudo aws s3 cp s3://BUCKET_NAME/nginx.conf .
```
<img width="559" height="110" alt="image" src="https://github.com/user-attachments/assets/a8ec2af1-bc9a-446e-841e-1c2d22277c1d" />

Then, restart Nginx with the following command:
```
sudo service nginx restart
```
To make sure Nginx has permission to access our files execute this command:
```
chmod -R 755 /home/ec2-user
```
And then to make sure the service starts on boot, run this command:
```
sudo chkconfig nginx on
```
Now when you plug in the public IP of your web tier instance, you should see your website, which you can find on the Instance details page on the EC2 dashboard. If you have the database connected and working correctly, then you will also see the database working. You’ll be able to add data. Careful with the delete button, that will clear all the entries in your database.

<img width="558" height="313" alt="image" src="https://github.com/user-attachments/assets/d06dd22a-8475-423e-8dca-178d3fdb10fc" />

<img width="557" height="232" alt="image" src="https://github.com/user-attachments/assets/8941fd01-d6a1-4ea1-959a-aeaac411fc96" />

20. Create image of Web Tier.

<img width="557" height="142" alt="image" src="https://github.com/user-attachments/assets/b92593e4-d898-4e92-96d0-d3627068a5b9" />

Create Target Group.

While the AMI is being created, we can go ahead and create our target group to use with the load balancer. On the EC2 dashboard navigate to Target Groups under Load Balancing on the left hand side. Click on Create Target Group.
<img width="546" height="453" alt="image" src="https://github.com/user-attachments/assets/33a15b53-7dee-4e31-80e3-d8438b5bc149" />

21. Internet Facing Load Balancer.

<img width="564" height="427" alt="image" src="https://github.com/user-attachments/assets/3b18617d-cc61-4221-9310-06e31f7c5eb9" />

<img width="597" height="227" alt="image" src="https://github.com/user-attachments/assets/75edcbf8-bc61-41c3-bc95-7b4ff6d7a2f1" />

22. create Launch Template.

Name the Launch Template, and then under Application and OS Images include the web tier AMI you created.

<img width="441" height="198" alt="image" src="https://github.com/user-attachments/assets/6f44508b-95af-4dd9-8890-6b4b965a57d5" />

<img width="486" height="492" alt="image" src="https://github.com/user-attachments/assets/3270f3d7-bdde-4997-b848-976bf5baee5d" />

<img width="521" height="145" alt="image" src="https://github.com/user-attachments/assets/6572f819-23f3-4949-8c74-662ef361f6e1" />

23. Create Auto Scaling group.

<img width="588" height="460" alt="image" src="https://github.com/user-attachments/assets/56f8e9df-ba95-46cd-9e67-0847444ffbdd" />

<img width="549" height="235" alt="image" src="https://github.com/user-attachments/assets/dcf9692b-ff16-4f25-a527-6659c7c8ca5f" />
<img width="541" height="266" alt="image" src="https://github.com/user-attachments/assets/d902f452-991a-49b6-b56b-69361da09dfc" />
<img width="564" height="119" alt="image" src="https://github.com/user-attachments/assets/f373cee0-cf57-470d-b04d-a7051e425cc0" />

You should now have your external load balancer and autoscaling group configured correctly. You should see the autoscaling group spinning up 2 new web tier instances. If you wanted to test if this is working correctly, you can delete one of your new instances manually and wait to see if a new instance is booted up to replace it. To test if your entire architecture is working, navigate to your external facing loadbalancer, and plug in the DNS name into your browser.

<img width="564" height="335" alt="image" src="https://github.com/user-attachments/assets/720190f7-6b3b-4315-a03c-ed2d41539b09" />






