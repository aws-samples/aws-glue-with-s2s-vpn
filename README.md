# Connecting AWS Glue to your On Premises Database Demo

This repository contains a demo showcasing features of AWS Services. When launching the CloudFormation template you will be able to see and test, how AWS Glue can connect to a Database in an On Premises environment through a Site-to-Site VPN connection.

## Requirements

* **AWS Account:** If you don’t have an account nor an account provided to you, [click here](https://aws.amazon.com/es/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc).
* **Text editor (optional):** Just to make things easier and more organised.

## Architecture

![AWS Glue Demo Architecture](/images/Demo-Architecture.jpg)
What you have here are two environments. One simulating the data center (DC-VPC) and on the right, the cloud environment (Cloud-VPC). 

In the Cloud VPC we have no internet connection, so all the communication is going to be private. In the private subnet we have the Elastic Network Interfaces created by AWS Glue to provide network connectivity for the service through your VPC. Note that the number of ENIs depends on the number of data processing units (DPUs) selected for an AWS Glue ETL job. AWS Glue DPU instances communicate with each other and with your JDBC-compliant database using ENIs.

The S3 endpoint is going to be used by AWS Glue to store temporary files and ETL scripts in S3. Also, in this demo, S3 is going to be our target for processed data coming from the Database located on the data center.

On the other hand, in the DC-VPC, we have an EC2 instance configured with OpenSwan software acting as the Customer Gateway Device. The router is going to reside in the public subnet and route all the private traffic coming from the Cloud to our private subnets. In these private subnets, we have a postgres database with the sample data already loaded. A private instance is created just to test the private connection to the cloud environment through AWS Site-to-Site VPN.

## Launch the template

1. Go to Amazon EC2 Console and on the left pane click on *Key Pairs* and create a key pair called **OpenswanKeyPair**. Choose the .pem format and create the key pair.

![Key_Pair](/images/KeyPair.png)

2. Deploy the AWS Cloud Formation template clicking on the button below:

[![Launch CFN stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://eu-west-1.console.aws.amazon.com/cloudformation/home?region=eu-west-1#/stacks/quickcreate?templateUrl=https%3A%2F%2Faws-glue-with-s2s-vpn.s3-eu-west-1.amazonaws.com%2FTemplates%2Fmain.yaml&stackName=aws-glue-with-s2s-vpn)

**(Optional)** Or deploy the template with CLI:

* If you don’t have the AWS CLI installed, follow [these](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) steps. And to configure the AWS CLI, follow [these](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config). 
* Clone the repository.
* Export the following parameters in your CLI:
```bash 
export AWSREGION=<YOUR AWS REGION>
export AWSPROFILE=<YOUR AWS PROFILE>
export STACKNAME=<THE NAME OF YOUR STACK>
```
* Go back to your terminal and create the CloudFormation stack:
```bash
aws cloudformation create-stack --stack-name $STACKNAME --template-url https://aws-glue-with-s2s-vpn.s3.amazonaws.com/Templates/main.yaml --tags Key=project,Value=glue-project --profile $AWSPROFILE --region=$AWSREGION --capabilities CAPABILITY_IAM
```
*NOTE*: The template takes 30 min to deploy approx.

## Explore your environment

You have two VPCs, one called Cloud-VPC which is the AWS side of the infrastructure and the other one called DC-VPC which is the simulated Data Center. 

**In the Cloud-VPC note that it has:**
* 4 subnets and all of them are private (even though there are two named as public, there is no internet gateway attached to the VPC so there is no public communication). The reason is because the communication between these two environments is going to be through the VPN, so there is no need to have public subnets nor internet access. 
* 1 private route table and inside you can see a S3 endpoint. This is for AWS Glue to access S3 privately (as this VPC does not has access to internet). 
* An EC2 instance with no public IP address. This is for you to test the VPN communication between the two environments.

**In the Data Center VPC:**
* Two private and two public subnets.
* One Nat Gateway.
* In the public subnet (AZ1) you have an EC2 instance already configured with OpenSwan software, just to serve as Customer’s Router (Customer Gateway Device).
* In the private subnet (AZ1) behind the router resides the postgres database to which AWS Glue is going to connect and also two EC2 instances, one called *PrivateInstanceToPostgresDCVPC* which is going to put the sample data inside of the DB and *DC-VPC-private-instance* which is going to be used to ping the Cloud-VPC instance to test VPN connection.

**AWS Site-to-Site VPN already created and fully configured:**
* Go to AWS VPC console and on the left pane, select Site-to-Site VPN connections and on the *Tunnel Details* tab check that only one tunnel is up. 
* The reason why only one tunnel is up is because of the OpenSwan configuration. This is not highly available and redundant but sufficient for this demo.

**In the AWS Glue console:**
* An AWS Glue Connection already configured with the JDBC connector to the postgres database.
* An AWS Glue Crawler which is used to scan and discover the database schema of your dataset and it is going to use the JDBC connection created.
* An AWS Glue database which is where the metadata of your dataset is going to be stored.

**Amazon S3 bucket:**
* There is a bucket called *demo-glues3bucket-RandomString* created by the template used to store the processed data by AWS Glue. 

## Test the Communication between the two environments

Go to AWS Systems Manager and on the left pane click on Managed Instances. Select *DC-Private-Instance* instance and click Actions > Start session. A new tab is opened with the instance’s terminal. Execute this command: 
``` 
ping <privateIP> 
```
where the private IP is the IP of the *Cloud-VPC-private-instance* instance.  

In my case the command looks like this:
```bash
ping 10.0.1.88
```
If you get this message when executing the command:

![Ping message](/images/ping_message.png)

That means that the VPN connection is working successfully and the communication is done privately.

## Test AWS Glue

Go to AWS Glue console:
* On the left pane click on *Connections*. You will see one connection listed called *glueConnection-(+random string*. Select it and click *Test connection*. Choose the role listed on the drop-down meno and test the connection. It may take a couple of minutes to receive the successful test message. 
* If the results is successful that means AWS Glue could connect to your On premises DB instance through the VPN connection.
* On the left pane click on *Crawlers*. Select the *glue-db-crawler* crawler and run it. This is going to scan the dataset, discover tables, and store the dataset schema in the *dbcrawler* AWS Glue database.
Wait until the following message is displayed:
`Crawler "glue-db-crawler" completed and made the following changes: 1 tables created, 0 tables updated. See the tables created in database dbcrawler.`
* Go and click in tables on the left pane and feel free to explore the dataset schema inside of the on premises database.

`CONGRATULATIONS!` The connection was done successfully and you have full connectivity between AWS Glue and your On Premises enviornment.

## Perform an AWS Glue Job (Optional)

Go to AWS Glue Console:
* On the left pane click on *Jobs*.
* Click on *Add Job*.
* Give the job a name and the select the role called *demo-Glue-(+random string)*. Leave the rest parameters as default and click next.

![Job1](/images/GlueJob_Config1.png)

* Select the data source *raw_sportstickets_public_players* and click next.
* In the next window, select *Change Schema* and click next.
* Selecr *Create tables in your data target*, SSelect *Amazon S3* as Data Store, *Parquet* as Format, and select the bucket that was created for you (the one that has *glue-demo-AccountId-Region* by name):

![Job2](/images/GlueJob_config2.png)

* Click save job and edit script
* In the next step click Run Job button and Run Job again
* On the right corner of console click the X button and wait for the job to finish

![Job3](/images/GlueJob_Config3.png)


* The Job has succeeded!! Now go to Amazon S3 console and click on the S3 bucket created by the template and expolore the procesed data


![Job4](/images/GlueJob_Config4.png)


* The files are loaded in Parquet format so all the ETL process is Done. `CONGRATULATIONS!`

## Clean up your environment

After completing the demo, delete AWS CloudFormation Stack using AWS Console or AWS CLI:
```bash
aws cloudformation delete-stack --stack-name $STACKNAME
```
**IMPORTANT:** If you did the optional part (*Perform an AWS Glue Job*) you will have to delete the resources created by you first before deleting the Cloud Formation stack:
1. Go to AWS Glue Console and on the lef pane click on Jobs and erase the prevous job created by you.
2. Go to EC2 console and on the letf pane click on Network Interfaces.
3. Select the ENI's used by your AWS Glue Job and delete all of them.
4. Now you can delete de AWS Cloud Formation template from the CLI or from the console.

Cheers!!

## Authors

* Daniel Neri
* Federica Ciuffo

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the LICENSE file.

