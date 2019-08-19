# Phoenix Application Solution

1. Create two Key Pairs for Application Server and MongoDB Server: 
   - Key Pair name for Applcation Server: nodejs
   - Key Pair name for MongoDBServer: mongo
   
   N.B. Download and save in a secure place.

2. Create 2 SNS Topic with name:
	a. CPU-Alarm
	b. Restart-App
	
3. Create S3Bucket "versioned" with name "lambda-functions-cloud-phoenix-kata" and upload two zip file with Lambda Functions (written in Python) "startNodeJSApp-ASG.py" and "restartNodeJSApp-ASG.py"	

4. Launch CF MongoDB Stack in region Ireland (download here: https://aws-quickstart.s3.amazonaws.com/quickstart-mongodb/templates/mongodb-master.template):
   - Open AWS Cloud Formation and click on "Create stack" --> Specify Template --> Upload a template file: mongodb-master.template
   
   Parameters:
      NETWORK CONFIGURATION (Nested VPC Stack)
      - Availability Zones
	  - Number of Availability Zones
	  - VPC CIDR
	  - Private Subnet 1 CIDR
	  - Private Subnet 2 CIDR
	  - Private Subnet 3 CIDR
	  - Public Subnet 1 CIDR
	  - Public Subnet 2 CIDR
	  - Public Subnet 3 CIDR
	  - Allowed Bastion External Access CIDR
	  
	  SECURITY CONFIGURATION 
	  - Key Pair Name 
	  
	  LINUX BASTION AMAZON EC2 CONFIGURATION (Nested Bastion Host Stack)
	  - Bastion AMI Operating System
	  - Bastion Instance Type
      - Number of Bastion Hosts
 
	  MONGODB DATABASE CONFIGURATION (Nested Database Configuration)
	  - Cluster Replica Set Count
      - IOPS
	  - MongoDB Versioning
	  - MongoDB Admin Username
	  - MongoDB Admin Password
	  - Node Instance Type
	  - Replica Shard Index
	  - Volume Size
	  - Volume Type
	  
   VERY IMPORTANT: insert SecurityGroupIngress rule to ALLOW TRAFFIC on port 27017 for all 3 Private Subnets.
   
5. Create "db" and test "testuser" + password "testpass" with folllowing instructions:
	
	- Connect in System Manager on Master DB Mongo node and each other node and digit following commands (for Amazon Linux 2)
	
	<Status MongoD Service>
	systemctl status mongod.service
    
	<If stopped, start MongoD Service>
	systemctl start mongod.service
	
	<Authentication like as admin and create testuser and testpass for dbtest>
	mongo
	db.auth(<MongoDB Admin Username>, <MongoDB Admin Password>)

	use dbtest
	db.createUser(
	  {
		user: "testuser",
		pwd: "testpass",
		roles: [
		   { role: "readWrite", db: "dbtest" }
		]
	  }
	)
	
6. Launch CF Application Server NodeJS Stack in region Ireland:
   - Open AWS Cloud Formation and click on "Create stack" --> Specify Template --> Upload a template file: cloud-phoenix-kata-AppServerInfrastructure.template
   
	Parameters:
      - VPC
	  - Private Subnet for Autoscaling Group
	  - Public Subnet for Application Load Balancer
	Resources:
      - LFstartNode: lambda function "startNodeJSApp-ASG" to start codepipeline and run command for new launched instance in Autoscaling Group
      - LFrestartNode: lambda function "restartNodeJSApp-ASG" to restart application after crash !! It's triggered when Crash-Alarm go on status "In Alarm" and it's send a Notification with SNS Topic 
      - LFRole: role for execution Lambda Function
      - SecurityGroup: security group SG-APPNODEJS for ALB and Launch Configuration
      - ALB: ALB "ALB-NodeJS" for balancing of the App Nodes. It balances on port 8080 (in HTTP)
      - ALBListener +  ALBListenerRule + ALBTG: the listener is set to forward HTTP Request (GET, HEAD) on port 8080 to Target Group (Autoscaling Group)
      - EC2-SSM-CW-ROLE + EC2SystemManagerCWProfile: EC2 instance Role and EC2 instance Profile to allow Session Manager and Run Command.
      - LC: setting Launch Configuration for Autoscaling Group (ImageId, InstanceType, KeyName, Security Groups, Userdata)
      - ASG: setting Autoscaling Group (AutoScalingGroupName, Cooldown, HealthCheckGracePeriod, HealthCheckType, VPCZoneIdentifier, TerminationPolicies, TargetGroupArn, MinSize, MaxSize)  to launch new instance (when it's necessary) in Private Subnet.
      - ASGPolicyScale: setting Target Tracking Policy to scale when the number of request are greater than 10 req /sec.
      - S3Bucket: S3 bucket with name "codedeploy-cloud-phoenix-kata-releases"  with Versioning enabled. This bucket is used to upload application code.
      - CPUAlarm: alarm to check CPU utilization of EC2 instances of the Autoscaling Group. The alarm is created when CPU Utilization is greater than 95%.
      - CrashAlarm: alarm to check HTTPCode_ELB_502_Count; when count is greater than 1, SNS sends a notify to RestartApp Topic and trigger Lambda Function restartNodeJSApp-ASG
      - ACWEventRule: cloud watch rule to launch Lambda Function when a node is launched with Autoscaling Group.
      - PermissionLambdaStartACWEventRule: permission for Lambda function startNodeJSApp-ASG
      - PermissionLambdaRestartSNS: permission for Lambda function restartNodeJSApp-ASG

7. Go to Lambda Function:
	  - in "startNodeJSApp-ASG": registering environment with Key: DB_CONNECTION_STRING and Value: mongodb://testuser:testpass@host1:27017,host2:27017,host3:27017/dbtest (host1,host2,host3 are IP address of the MongoDB nodes) and SAVE IT.
	  - in "restartNodeJSApp-ASG": registering environment with Key: DB_CONNECTION_STRING and Value: mongodb://testuser:testpass@host1:27017,host2:27017,host3:27017/dbtest (host1,host2,host3 are IP address of the MongoDB nodes) and SAVE IT.

8. For Continuos Delivery, it's necessary create a zipped folder cloud-phoenix-kata and create:
	- a folder with Name: File (with application code)
	  
	- a file with Name: appspec.yml (used from CodeDeploy Agent to upload application code)
		  
    On S3 Bucket codedeploy-cloud-phoenix-kata-releases, create a folder with name cloud-phoenix-kata and upload cloud-phoenix-kata.zip that contains App Node JS code
	
9. Create bucket cloud-phoenix-kata-repository (for CodePipeline)

10. Launch CF CodeDeploy + CodePipeline Stack in region Ireland.
	- Open AWS Cloud Formation and click on "Create stack" --> Specify Template --> Upload a template file: cloud-phoenix-kata-CodeDeployPipeline.template
	
	Resources:
	  - Application: creating application in CodeDeploy
	  - DeploymentGroup: creating Deployment Group (DeploymentGroupName, AutoScalingGroups) for created application.
	  - AppPipeline: creating Codepipeline for continuos delivery of the application 
	  - AmazonCloudWatchEventRule: cloud watch to allow starting CodeDeploy when an operation PutObject in S3 is issued.
	  - AmazonCloudTrail: setting a cloud trail for S3 bucket when you will upload application
	  - PolicyCW + CWRole: policy and role for Cloud Watch Event Rule
	  - CDRole: policy and role for CodeDeploy
	  - PolicyCP + CPRole: policy and role for CodePipeline
	  - BucketLogsCT: bucket for log Cloud Trail
	  - BucketPolicyLogsCT: bucket policy for log Cloud Trail
		
11. Modify Desired Number Instance on the Auto Scaling Group to 1 --> STARTING LAMBDA FUNCTION "startNodeJSApp-ASG1.py"

	- Navigate to URL: http://alb-dns-name:8080/ 
	
 	[Optional]: create a Hosted Zone (in example cloud-phoenix-kata.local) and inser a CNAME www with value ALB DNS Name
		 When the user execute a HTTP Request to http://www.cloud-phoenix-kata.corp:8080/ it will receive the Express page. 
		 
12. CPU Alarm: Send Notify SNS CPU-Alarm
	- Possible Test: 
		sudo apt-get install stress
		stress --cpu 2 --timeout 300
		
13. After Crash: send Notify SNS RestartApp and Start Lambda restartNodeJSApp-ASG
	- Possible Test: navigate to URL: http://alb-dns-name:8080/crash
		
14. AutoScale: autoscale when the number of the requests is grether than 10req/sec.
	- Possilbe Test: go to https://loadimpact.com/ and insert alb-dns-name:8080
	
For Backup, it's possible read some solutions in this link: https://docs.mongodb.com/manual/core/backups/
