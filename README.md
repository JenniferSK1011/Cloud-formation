# Cloud-formation
The provided YAML CloudFormation template defines a Lab environment in AWS, including the creation of a VPC, private subnet, Internet Gateway, and associated resources such as route tables, security groups, and an EC2 instance.
Components of the Template
Parameters:

LabVpcCidr: CIDR block for the VPC (default: 10.0.0.0/20).
PrivateSubnetCidr: CIDR block for the private subnet (default: 10.0.0.0/24).
AmazonLinuxAMIID: Fetches the latest Amazon Linux 2 AMI ID using SSM Parameter Store.
Resources:

VPC (LabVPC):
Creates a Virtual Private Cloud (VPC) with DNS support and hostnames enabled.
Internet Gateway (IGW):
Creates an Internet Gateway and attaches it to the VPC via VPCtoIGWConnection.
Route Table (PrivateRouteTable):
A private route table associated with the private subnet.
Private Subnet (PrivateSubnet):
Creates a private subnet within the VPC in the first Availability Zone of the region.
Route Table Association:
Associates the private subnet with the private route table.
Security Group (AppSecurityGroup):
Allows inbound SSH traffic (port 22) from any IP address.
EC2 Instance (Instance):
Launches a t3.micro instance in the private subnet using the Amazon Linux 2 AMI.
Outputs:

LabVPCDefaultSecurityGroup: Outputs the default security group ID of the VPC.
Notes and Suggestions:
Private Subnet without NAT Gateway: Since this setup only includes an Internet Gateway but no NAT Gateway, instances in the private subnet cannot access the internet directly. Consider adding a NAT Gateway if internet access is required for private instances.

Secuty Group Configuration: Allowing SSH (22) from 0.0.0.0/0 is potentially insecure. Use a more restrictive CIDR block, such as your public IP range, for better security.

IAM Role for EC2: To follow best practices, add an IAM Role with least privilege access for the EC2 instance, especially if it needs to communicate with AWS services.

Public Subnet Option: If a public subnet is needed for better connectivity, you can add one with its own route table pointing to the Internet Gateway.







yaml
Copy code
AWSTemplateFormatVersion: '2010-09-09'
Description: Lab VPC with Private Subnet and Internet Gateway

Parameters:
  LabVpcCidr:
    Type: String
    Default: 10.0.0.0/20

  PrivateSubnetCidr:
    Type: String
    Default: 10.0.0.0/24

  AmazonLinuxAMIID:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Resources:
  # VPC
  LabVPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: !Ref LabVpcCidr
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: Lab VPC

  # Internet Gateway
  IGW:
    Type: 'AWS::EC2::InternetGateway'
    Properties:
      Tags:
        - Key: Name
          Value: Lab IGW

  VPCtoIGWConnection:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      InternetGatewayId: !Ref IGW
      VpcId: !Ref LabVPC

  # Private Route Table
  PrivateRouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref LabVPC
      Tags:
        - Key: Name
          Value: Private Route Table

  # Private Subnet
  PrivateSubnet:
    Type: 'AWS::EC2::Subnet'
    Properties:
      VpcId: !Ref LabVPC
      CidrBlock: !Ref PrivateSubnetCidr
      AvailabilityZone: 
        Fn::Select:
          - 0
          - Fn::GetAZs: !Ref AWS::Region
      Tags:
        - Key: Name
          Value: Private Subnet

  PrivateRouteTableAssociation:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      SubnetId: !Ref PrivateSubnet

  # Security Group
  AppSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: Enable access to App
      VpcId: !Ref LabVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: App Security Group

  # EC2 Instance
  Instance:
    Type: 'AWS::EC2::Instance'
    Properties:
      InstanceType: t3.micro
      ImageId: !Ref AmazonLinuxAMIID
      SubnetId: !Ref PrivateSubnet
      SecurityGroupIds:
        - !Ref AppSecurityGroup
      Tags:
        - Key: Name
          Value: App Server

Outputs:
  LabVPCDefaultSecurityGroup:
    Value: !GetAtt LabVPC.DefaultSecurityGroup
    Description: Default Security Group ID of the VPC
