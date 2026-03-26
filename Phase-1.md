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

