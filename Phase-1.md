# Starting point
This is my phase 1 report for my AWS Project.
## Plan
I have a plan to create EC2 instance to host a website and a Bastion Host to securely connect to it from my PC.

## Implementation
### Starting
First i'll be downloading and installing AWS CLI to my windows PC so I can do everything via CLI. And then I will connect to the learner lab with the credentials found from the `AWS Details` panel on the learner lab.
And then with command `aws sts get-caller-identity` I'm able to confirm that I'm in 



<img width="1010" height="128" alt="Näyttökuva 2026-03-26 135029" src="https://github.com/user-attachments/assets/2496ca15-4f4d-4a28-9b61-3586e045f7d4" />

And now to start the real work on the project, first thing we need is a VPC, which can be created with command `aws ec2 create-vpc --cidr-block 10.0.0.0/16`. Note up the VPC-ID.

Then we create subnets 1 (public) `aws ec2 create-subnet --vpc-id <VPC-ID> --cidr-block 10.0.1.0/24`  
and 2 (private) `aws ec2 create-subnet --vpc-id <VPC-ID> --cidr-block 10.0.2.0/24`. Note up the subnet-IDs.

Next we create Internet Gateway `aws ec2 create-internet-gateway`,  
and attach it to the VPC `aws ec2 attach-internet-gateway --vpc-id <VPC-ID> --internet-gateway-id <InternetGateway-ID>`.

Next step is to create a route table `aws ec2 create-route-table --vpc-id <VPC-ID>` and note up the route table id.  
Then create a route to the Internet `aws ec2 create-route --route-table-id <RouteTableID> --destination-cidr-block 0.0.0.0/0 --gateway-id <InternetGatewayID>`.  
And associate the public subnet to the route. `aws ec2 associate-route-table --route-table-id <RouteTableID> --subnet-id <SubnetID>`. 

### Creating Security Groups

Then we create security groups for the Bastion Host and the Web Server.

Bastion security group:

```
aws ec2 create-security-group \
  --group-name bastion-sg \
  --description "Bastion access" \
  --vpc-id <VPC-ID>

``` 
```
aws ec2 authorize-security-group-ingress \
  --group-id <bastionGroupID> \
  --protocol tcp \
  --port 22 \
  --cidr <Your own IP/32>
```

Web Server security group:

```
aws ec2 create-security-group \
  --group-name webserver-sg \
  --description "Web server" \
  --vpc-id <VPC-ID>
```
```
aws ec2 authorize-security-group-ingress \
  --group-id <webserverGroupID> \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
```
```
aws ec2 authorize-security-group-ingress \
  --group-id <webserverGroupID> \
  --protocol tcp \
  --port 22 \
  --source-group <bastionGroupID>
```
### Launching instances

First the Bastion instance

```
aws ec2 run-instances \
  --image-id ami-02dfbd4ff395f2a1b \
  --count 1 \
  --instance-type t2.micro \
  --key-name vockey \
  --security-group-ids <bastionGroupID> \
  --subnet-id <PublicSubnetID> \
  --associate-public-ip-address
```

Web server Instance:

```
aws ec2 run-instances \
  --image-id ami-02dfbd4ff395f2a1b \
  --count 1 \
  --instance-type t2.micro \
  --key-name vockey \
  --security-group-ids <webserverGroupID> \
  --subnet-id <PrivateSubnetID>
```

### Connecting to the instances

Now that the instances are running, we are able to use PuTTY to connect to the Bastion Host using it's public IP-Address. 
And from the Bastion Host, we are able to connect to the Web Server instance via SSH, but first we need to transfer the ssh key used from local machine to Bastion Host.
That can be done with PuTTY Secure Copy Client.   
`pscp -i key <key.ppk> <key.pem> ec2-user@<BastionHostPublicIP>:/home/ec2-user/`

Then before you are able to use the key, you have to modify it's owner rights with `chmod 400 <key.pem>`.

And then the ssh connection from Bastion Host to Web Server works.

<img width="550" height="195" alt="image" src="https://github.com/user-attachments/assets/f31514fc-5875-4f6f-b2b4-6a62def340e1" />

# Phase 2

## Installing Apache

Now I'm connected to the Web Server via the Bastion host, and I try to install Apache, but the only allowed trafic is the SSH from the Bastion host. So I need to enable traffic from the internet to the Web Server, either by putting the Web Server into the public subnet, or by creating a NAT Gateway. I decide to atleast try creating the NAT Gateway first.

# NAT Gateway

We first need elastic IP address for the gateway, to obtain it I must use command `aws ec2 allocate-address` which outputs the public IP and allocationID needed shortly. Then we create the NAT Gateway with command `aws ec2 create-nat-gateway --subnet-id <publicsubnet-id> --allocation-id <allocation-id>` and wait couple minutes for the NAT Gateway to start up. Then we create a new route table for the Private subnet `aws ec2 create-route-table --vpc-id <VPC-ID>` then we create a new route `aws ec2 create-route --route-table-id <routetableID> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <NATGatewayID>` using the routetable we just created.

Then we associate the route table with the Private subent `aws ec2 associate-route-table --subnet-id <PrivateSubnetID> --route-table-id <RouteTableID>`. And now we are able to connect to the internet on the Web Server.

<img width="449" height="84" alt="image" src="https://github.com/user-attachments/assets/c5cbba68-c108-4d9d-b40c-8b40a8933009" />

# Apache

Now trying to install apache with `sudo yum install -y httpd` and now it works. Next I move a html site I've created into the /var/www/html/ folder.

Now i can curl localhost, to see the site, but to access it from the internet, i need to configure a Load Balancer to act as an entry point.

# Load Balancer

First we need to create a target group to configure where the load balancer sends the traffic. `aws elbv2 create-target-group --name <targetgroupname> --protocol HTTP --port 80 --vpc-id <VPC-ID> --target-type instance`. The results give us TargetGroupArn which we note up.

Then we need to register the Web Server as a target `aws elbv2 register-targets --target-group-arn <TargetGroupArn> --targets ID=<WebServerID>`.
Then we also need to create a Load Balancer Security Group, `aws ec2 create-security-group --group-name LB-SG --description "Load balancer security group" --vpc-id <VPC-ID>` and then add a rule `aws ec2 authorize-security-group-ingress --group-id <LB-SG ID> --protocol tcp --port 80 --cidr 0.0.0.0/0`.

We also need to create another public subnet, for the load balancer to actually balance the traffic. `aws ec2 create-subnet --vpc-id <VPC-ID> --cidr-block 10.0.3.0/24 --availability-zone <us-east-1a>` and enable automatic public ip on both public subnets `aws ec2 modify-subnet-attribute --subnet-id <subnet-ID> --map-public-ip-on-launch`. 

Then we create the load balancer, `aws elbv2 create-load-balancer --name <loadbalancername> --subnets <publicsubnet1> <publicsubnet2> --security-groups <LB-SG ID> --scheme internet-facing --type application` note the loadbalancerarn. 

Next we create a listener `aws elbv2 create-listener --load-balancer-arn <LoadBalancerArn> --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=<TargetGroupArn>`. Then with command `aws elbv2 describe-load-balancers` we can retrieve the DNSname which can be used to fetch the website
<img width="2522" height="1134" alt="image" src="https://github.com/user-attachments/assets/4b21a34e-af76-413e-97fb-eccc3d4de0fb" />



https://github.com/user-attachments/assets/7db8a1c5-c2d7-491c-aa38-5977fc731dcf





## CloudWatch CPU Utilization

Next I'm going to configure CloudWatch alarm to notify me when the CPU utilization goes above 70%, first we need to create SNS topic `aws sns create-topic --name <alertname>` and then subscribe to it with `aws sns subscribe --topic-arn <TopicArn> --protocol email --notification-endpoint <emailAddress>`. Then we create the alarm `aws cloudwatch put-metric-alarm --alarm-name "CPUHigh" --metric-name CPUUtilization --namespace AWS/EC2 --statistic average --period 300 --threshold 70 --comparison-operator GreaterThanThreshold --evaluation-periods 2 --dimensions Name=InstanceId,Value=<InstanceID> --alarm-actions <TopicArn>`.

<img width="406" height="78" alt="image" src="https://github.com/user-attachments/assets/43bedf57-0d6d-43e8-ad3b-3427f9df5867" />
<img width="520" height="176" alt="image" src="https://github.com/user-attachments/assets/2648537a-c06d-483f-a814-acfa86c9ff8d" />



