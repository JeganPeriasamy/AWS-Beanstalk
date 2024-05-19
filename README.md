AWS SERVICES USED
<img src="https://cdn.hashnode.com/res/hashnode/image/upload/v1686905570005/3ec0e8ee-d811-430b-b320-b8705be620be.png?w=1600&h=840&fit=crop&crop=entropy&auto=compress,format&format=webp" alt="" >
# A Complete Architecture Setup of AWS Servies to handle the Request
# How it works ?
When the User hits the URL - it seaches the Data in the CloudFront, it sends it data is available or request moves to the Beanstalk , and it controls the request by LoadBalancer , also monitored by cloudWatch, then the request goes to the MQ messager Service then after waiting its DATA is fetched from the RDS and response is send to User
# Frontend 	
Beanstalk 
1.	VM for Tomcat
2.	NGINX Load balancer 
3.	Auto scaling
4. S3 /EFS – Storage 	

# Backend 
1. RDS Instance (Database) 
2. ELASTIC CHACHE
3. ACTIVE MQ
4. ROUTE 53 
5. CLOUDFRONT 

# Procedure 
1.	Create KEY PAIR – For : Beanstalk instance Login
2.	Create SG – For: Elasticcache ,RDS, and ActiveMQ
3.	Create – Elasticcache ,RDS, and ActiveMQ
4.	Create Elastic beanstalk Environment 
5.	Update SG for Backend to allow traffic from Bean SG 
6.	Update SG for Backend to allow Internal traffic 
7.	Launch Ec2 for DB Initializing – then from there we will do a MySQL login to our RDS and initialize DB
8.	Login into EC2 and initialize DB
9.	Change Healthcheck on beanstalk to /login
10.	Add 443 https listener to ELB
11.	Build Artifact with Backend Information 
12.	Deploy the artifact to Beanstalk 
13.	Create CDN ( Content Delivery Network ) with ssl certification for HTTPS Connection
14.	Update Endpoint  in GoDaddy DNS Zones  or In R53 PUBLIC DNS Zone
15.	Test the URL


# STEPS 
1.	Create KEYPAIR ---- Name:vprofilebeankey------PEM
2.	Create SG (Name:vprofilebackendSG = All Traffic —its own SG ( To interact with  the services that uses this)
3.	RDS
4.	subnet group ----Create---name:vprofilerdssubgroup---Default VPC---Add All AZ-and Subnets--- ( NETWORK THAT WE CREATED AND This NETWORK YOU ARE GOING TO LAUNCH YOUR DATABASE INSTANCE )
5.	PARAMETER GROUPS ---- ( TO CHANGE THE DATABASE PARAMETER AND SINCC RDS YOU CANT SO SSH AND CHANGE CONFIGURATION -  FOR THE DATABASE YOU WANT TO CREATE )
 Create parameter group ---MYSQL8----Name:vprofileparagroup---create 
6. Cate database---Standard create---MYSQL---Templete:Dev/Test---Availability:Single db instance----Setting –name:vprofilerdssql---username:Admin---Autogenerate password----burstable classes---show old select T2 micro or T3----storage:Gp2---20GB---Connectivity: Don’t connect to Ec2----No public access ----Security group: Backend we created---Additional cnfig: Initial database name = accounts----parameter group:ours---logs select all logs to cloudwatch----CREATE
7. View credential detail : copy and save user name and password

8. ELASTIC CHACHE 
      •	Create Subnet group----Name:vprofilememchachedsg---all the zones are created by default---Create
      •	Create parameter group --- Name:vprofilememchachedpg---Version:memchache 1.6 ---Create 
      •	Dashboard ---Create memechache cluster ---Aws cloud---Name:vprofileelasticchsvc----parameter group: select we created ----Node type:cache t2 micro----nodes:1----select our SG -Create 
      •	Copy the endpoint and save 

9.	MQ ( Fully managed message broker service )
      •	Create ---RabbitMQ---Single instance ----Name:vprofilermq---t3micro----Username:rabbit , Password:Blue7890bunny------Copy and save it -----Network and Security : Private----SG:Backend ----Create 
10.	Go to github---SRC---MAIN---resources ---dbbackup.sql----


Note : ( THE ONLY WAY TO ACCESS OUR INSTANCE, WHICH IS IN PRIVATE SUBNETS IS THROUGH THE SAME VPC----IN  OUR INSTANCE)-----SO WE ARE GOING TO LAUNCH INSTANCE IN THA SAME VPC ,so we can connect our instance and run this sql query)


11.	EC2 ---MAKE SURE IN SAME REGION ---Name:MySQLClient----OS:UBUNTU----select the key pair----Network setting Edit ---Create SG –Name:mysqlclientsg (ssh ansd my ip)---create
12.	WE HAVE TO ENABLE THE BACKEND SG TO ALLOW CONNECTION FROM THE INSTANCE SG TO SG
13.	Go to MysqLclientsg----copy the GS ID ---go to Backend SGInbound rules :add (Custom TCP/3306/mysqlsg)
14.	Copy the Public IP OF EC2----open git bash ----
•	ssh  -I Downloads/ vprofilebeankey.pem ubuntu@Public Ip
•	sudo apt update
•	sudo apt install mysql-client –y
(If you used Amazon Linux or CentOS then need to install Mariadb)
•	mysql –h endpoint –u admin –ppassword accounts
•	show tables;
•	exit
•	git clone githubcode
•	ls
•	cd vprofile-project/
•	ls
•	ls src/main/resources/db_backup.sql
To Execute all these sql queries on the database accounts which is on RDS
•	mysql –h endpoint –u admin –ppassword accounts  < src/main/resources/db_backup.sql
•	mysql –h endpoint –u admin –ppassword accounts 
•	show tables;
•	exit
Now we can delete the instance because the RDS and instance ready to connect
•	go to MQ---Copy th endpoint---remove the port number and paste----
•	Elasticchache ---Configure endpoint copy ---remove the port number and //upto and paste


# NOW THE COMPLTE BACKEND IS READY -------------------------------------------------------------------------------

# AWS Beanstalk SetUp
1.	It assumes some role ----but it has issues so we create the role 
2.	IAM Roles-- Aws services-EC2policy permission:Elasticbeanstalkwebtier , Admin access elasticbean , elasticbean SNS , Elastic bean custom platform ec2 role--name: vprofilebeanrole --- CREATED - DELETE THE Elastic beanstalk service role 
3.	BEANSTALK - CREATE- Webserver environment - Application name: vprofile-app - environ name:vprofile-app-prod-- Domain : vprofileapp12 - Platform: Tomcat , branch: 11 --- presets: Custom config - HERE WE  HAVE 2 ROLES (SERVICE ROLE AND EC2 ROLE )-- for ec2 select we created  service create it and new -- keypair:we created = default vpc- Activate the Public IP  Address and select all subnets - Tags: name,vproapp and project,vprofile--- Capacity : Load balanced ,ASG (Min:2 , Max:8 ) -- 
Instance type : keep only t3 micro --Deployment policy: rolling and percentage:50-- LAUNCH IT MY BOY
4.	While clicking the Domain in created one you can see your application you created

Note: 
1.	( SCALING TRIGGER : Network Out ---more user more network out)
2.	Rolling updates and deployment policy --- 
•	All at once = all instance at the same time brought up and down
•	Rolling = slowly replaces previous versions of an application with new versions of an application by completely replacing the infrastructure on which the application is running.
•	Rolling with additional batch : without lag you want to do , it  deploy extra instance at the time of upgrade
•	Immutable: Safe and most expensive ---launch all new 
•	Traffic splitting : Deploy the new version to a fresh group of instances and temporarily split incoming client traffic between the existing application version and the new one

WE are going to do:
1.	Enable ACL on S3 Bucket
2.	Update the health check in the Target group
3.	Update SG

Enable ACL on S3 Bucket
S3To Know which cross check with region  Select the bucket permission  object ownership edit   enable ACL  ( To : account access to the s3 without using of policy we are using it – OWNER OF THE BUCKET WHO CREATED THE WRITER OF THE BUCKET HAS ACCESS TO THE S3)  SAVE

Update the health check in the Target group
Beanstalk - our environment configuration  Instance traffic and scalingprocess – action –edit –health check (\login), Enable session stickiness –save

Add https Listener (add Port 443 https listener and select SSL Certificate from ACM)
1.	Beanstalk - our environment configurationlisteneradd (Port:443 ,HTTPS,ACM Certificate ) and save and come down give apply 
2.	Once you apply – going to change the target group and health will become ok to severe 
Update SG 
1.	ec2 -2 are running SG-> copy the BEANSTALK instance sg id ( by going in )find the vprofilebackend SG and go to inbound rule  add rule ( All traffic and sg id , Description: Allow all traffic from instance SG ) and (Cusrtom tcp,3306, sg id) , (Cusrtom tcp,11211, sg id) , (Cusrtom tcp,5672, sg id)

2.	WHY THIS? 

•	ALL OUR BACK END SERVICES ARE IN BACKEND SG , INSTANCE OF BEANSTACK WILL ACCESS BACKEND SERVICES AND IT IS GOING TO PASS THROUGH THE BACKEND SECURITY GROUP AND IT WILL ACCESS ON PORT (RDS:3306 , ELASTICCHACHE:11211 AND MQ : 5672 ) 
•	In order to allow all those port number access from the instance 

BUILD AND DEPLOY ARTIFACT 
CLONE GIT TO LOCAL and CHANGE THE DATA OF THIS PROJECT
1.	Open git bash - mkdir /e/aws-refactor 
2.	Open VSC  file—new window –copy the url of git and clone 
3.	After cloned
4.	srcmainresources-application.properties file open
5.	remove the db01  and give the endpoint of rds and change the password you saved 
6.	remove the mc01  and give the endpoint of memcache and no port number at the end of url 
7.	Remove the rmq01 and give the endpoint of MQ and 5671 (CHECK WITH IT PORT )and change the username:rabbit and password:blu.. you saved 
8.	save it

BUILD 
1.	go to … on up  terminal  new (Git bash) (   MAKE SURE THAT U ARE IN THE SAME FOLDER ) 
2.	cat src/main/resources/application.properties
3.	mvn –version
4.	if any problem in this lunch Ubuntu VM and do
5.	mvn install
6.	ls
7.	we should have the target here ( vprofile.v2.war)
8.	Go to beanstalk  our environment we created - create and deploy - Choose file  select the ( vprofile.v2.war ) from the saved folder of our computer -change version:vprofile-app-v.2.5  deploy 
9.	ITS TAKES TIME TO COMPLETE
10.	Check the url of the domain- it shows - change into https//  shows not secure because our certificate is for different domain 

11.	Go to godaddy  manage  create record  (CNAME /vprofile/end point or doamin of beanstalk)save

12.	https://vprofile.jegpr.shop---it works 

13.	Login into the website 

•	username : admin_vp
•	pass: admin_vp

14.	If any error it shows 
Reason: 
•	DATABASE NOT CONNECTED PROPERLY
•	SO NEED TO CHECK application.properties FILE DETAILS 
•	CHECK THE BACKEND SECURITY GROUP ( ALL TRAFFFIC OR PORT )
•	IF ANY CHANGES ( SAVE  BUILD  DEPLOY IT )
•	IF AFTER GOING IN CHACHE ISSUE ITS ALSO SG ISSUE 

CLOUDFRONT (  CONTENT DELIVERY NETWORK ) 

1.	create  our domain name : vprofile.jegpr.shop , Protocal : Matchviewer  Viewer: allow HTTP Methods (all get put …)  setting : all location, ssl certificate , security policy : TLSv1 - CREATE 
2.	Deploying ----it takes long time
3.	https://vprofile.jegpr.shop---it works ---F12 - TO SEE WHERE IT COMES FROM  


IT WORKS!!!!!!
Links:
https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.rolling-version-deploy.html 
