# AWS CloudFormation

## Giới thiệu CloudFormation

### CloudFormation là gì?

AWS CloudFormation là Infrastructure as Code service của AWS, cho phép định nghĩa và provision AWS resources bằng JSON hoặc YAML templates.

**Ưu điểm:**
- Infrastructure as Code
- Version control cho infrastructure
- Repeatable và consistent
- Rollback tự động khi lỗi
- Quản lý dependencies giữa resources
- Free (chỉ trả tiền cho resources tạo ra)

**Concepts:**
- **Template**: File JSON/YAML định nghĩa resources
- **Stack**: Group của resources được tạo từ template
- **Change Set**: Preview changes trước khi apply
- **StackSet**: Deploy stacks across multiple accounts/regions

### Template Structure

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Template description

Parameters:
  # Input parameters

Mappings:
  # Static variables

Conditions:
  # Conditional logic

Resources:
  # AWS resources (required)

Outputs:
  # Output values
```

## Basic Template

### EC2 Instance

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Simple EC2 instance

Parameters:
  InstanceType:
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - t2.small
      - t2.medium
    Description: EC2 instance type
  
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
    Description: EC2 key pair name

  SSHLocation:
    Type: String
    Default: 0.0.0.0/0
    Description: IP range for SSH access
    AllowedPattern: '(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/(\d{1,2})'

Mappings:
  RegionMap:
    us-east-1:
      AMI: ami-0c55b159cbfafe1f0
    us-west-1:
      AMI: ami-0d5075a8dfa4e5ea4
    eu-west-1:
      AMI: ami-0ea3405d2d2522162

Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: !FindInMap [RegionMap, !Ref 'AWS::Region', AMI]
      KeyName: !Ref KeyName
      SecurityGroups:
        - !Ref InstanceSecurityGroup
      Tags:
        - Key: Name
          Value: MyWebServer
      UserData:
        Fn::Base64: |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "<h1>Hello from CloudFormation</h1>" > /var/www/html/index.html
  
  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable SSH and HTTP
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: !Ref SSHLocation
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

Outputs:
  InstanceId:
    Description: EC2 Instance ID
    Value: !Ref EC2Instance
  
  PublicIP:
    Description: Public IP address
    Value: !GetAtt EC2Instance.PublicIp
  
  WebsiteURL:
    Description: Website URL
    Value: !Sub 'http://${EC2Instance.PublicDnsName}'
```

### Deploy Template

```bash
# Create stack
aws cloudformation create-stack \
  --stack-name my-web-server \
  --template-body file://ec2-template.yaml \
  --parameters \
    ParameterKey=InstanceType,ParameterValue=t2.micro \
    ParameterKey=KeyName,ParameterValue=my-key

# List stacks
aws cloudformation list-stacks

# Describe stack
aws cloudformation describe-stacks \
  --stack-name my-web-server

# Get outputs
aws cloudformation describe-stacks \
  --stack-name my-web-server \
  --query 'Stacks[0].Outputs'

# Update stack
aws cloudformation update-stack \
  --stack-name my-web-server \
  --template-body file://ec2-template-v2.yaml \
  --parameters ParameterKey=InstanceType,ParameterValue=t2.small

# Delete stack
aws cloudformation delete-stack \
  --stack-name my-web-server

# Wait for completion
aws cloudformation wait stack-create-complete \
  --stack-name my-web-server
```

## VPC with Subnets

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC with public and private subnets

Parameters:
  EnvironmentName:
    Type: String
    Default: Production
    Description: Environment name prefix
  
  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
    Description: VPC CIDR block
  
  PublicSubnet1CIDR:
    Type: String
    Default: 10.0.1.0/24
  
  PublicSubnet2CIDR:
    Type: String
    Default: 10.0.2.0/24
  
  PrivateSubnet1CIDR:
    Type: String
    Default: 10.0.11.0/24
  
  PrivateSubnet2CIDR:
    Type: String
    Default: 10.0.12.0/24

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-VPC
  
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-IGW
  
  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC
  
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Ref PublicSubnet1CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-Public-Subnet-AZ1
  
  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Ref PublicSubnet2CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-Public-Subnet-AZ2
  
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Ref PrivateSubnet1CIDR
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-Private-Subnet-AZ1
  
  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Ref PrivateSubnet2CIDR
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-Private-Subnet-AZ2
  
  NatGateway1EIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
  
  NatGateway2EIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
  
  NatGateway1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGateway1EIP.AllocationId
      SubnetId: !Ref PublicSubnet1
  
  NatGateway2:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGateway2EIP.AllocationId
      SubnetId: !Ref PublicSubnet2
  
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-Public-Routes
  
  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  
  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1
  
  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2
  
  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-Private-Routes-AZ1
  
  DefaultPrivateRoute1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway1
  
  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      SubnetId: !Ref PrivateSubnet1
  
  PrivateRouteTable2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-Private-Routes-AZ2
  
  DefaultPrivateRoute2:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway2
  
  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      SubnetId: !Ref PrivateSubnet2

Outputs:
  VPC:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub ${EnvironmentName}-VPCID
  
  PublicSubnets:
    Description: Public subnets
    Value: !Join [',', [!Ref PublicSubnet1, !Ref PublicSubnet2]]
    Export:
      Name: !Sub ${EnvironmentName}-PUB-NETS
  
  PrivateSubnets:
    Description: Private subnets
    Value: !Join [',', [!Ref PrivateSubnet1, !Ref PrivateSubnet2]]
    Export:
      Name: !Sub ${EnvironmentName}-PRIV-NETS
```

## Application Load Balancer

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Application Load Balancer with Auto Scaling

Parameters:
  EnvironmentName:
    Type: String
    Default: Production
  
  KeyName:
    Type: AWS::EC2::KeyPair::KeyName
  
  InstanceType:
    Type: String
    Default: t2.micro
  
  MinSize:
    Type: Number
    Default: 2
  
  MaxSize:
    Type: Number
    Default: 6
  
  DesiredCapacity:
    Type: Number
    Default: 2

Mappings:
  AWSRegionToAMI:
    us-east-1:
      AMI: ami-0c55b159cbfafe1f0
    us-west-2:
      AMI: ami-0d1cd67c26f5fca19

Resources:
  LoadBalancerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Load Balancer Security Group
      VpcId:
        Fn::ImportValue: !Sub ${EnvironmentName}-VPCID
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
  
  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web Server Security Group
      VpcId:
        Fn::ImportValue: !Sub ${EnvironmentName}-VPCID
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref LoadBalancerSecurityGroup
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
  
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: !Sub ${EnvironmentName}-ALB
      Subnets:
        Fn::Split:
          - ','
          - Fn::ImportValue: !Sub ${EnvironmentName}-PUB-NETS
      SecurityGroups:
        - !Ref LoadBalancerSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-ALB
  
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: !Sub ${EnvironmentName}-TG
      Port: 80
      Protocol: HTTP
      VpcId:
        Fn::ImportValue: !Sub ${EnvironmentName}-VPCID
      HealthCheckEnabled: true
      HealthCheckPath: /
      HealthCheckProtocol: HTTP
      HealthCheckIntervalSeconds: 30
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3
      Matcher:
        HttpCode: 200
      TargetGroupAttributes:
        - Key: deregistration_delay.timeout_seconds
          Value: '30'
  
  ALBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP
  
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: !Sub ${EnvironmentName}-LaunchTemplate
      LaunchTemplateData:
        ImageId: !FindInMap [AWSRegionToAMI, !Ref 'AWS::Region', AMI]
        InstanceType: !Ref InstanceType
        KeyName: !Ref KeyName
        SecurityGroupIds:
          - !Ref WebServerSecurityGroup
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd
            systemctl start httpd
            systemctl enable httpd
            echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: !Sub ${EnvironmentName}-WebServer
  
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: !Sub ${EnvironmentName}-ASG
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: !Ref MinSize
      MaxSize: !Ref MaxSize
      DesiredCapacity: !Ref DesiredCapacity
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300
      VPCZoneIdentifier:
        Fn::Split:
          - ','
          - Fn::ImportValue: !Sub ${EnvironmentName}-PRIV-NETS
      TargetGroupARNs:
        - !Ref TargetGroup
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-ASG-Instance
          PropagateAtLaunch: true
  
  ScaleUpPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      AutoScalingGroupName: !Ref AutoScalingGroup
      Cooldown: 60
      ScalingAdjustment: 1
  
  ScaleDownPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AdjustmentType: ChangeInCapacity
      AutoScalingGroupName: !Ref AutoScalingGroup
      Cooldown: 60
      ScalingAdjustment: -1
  
  CPUAlarmHigh:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Scale up if CPU > 70%
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 70
      AlarmActions:
        - !Ref ScaleUpPolicy
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AutoScalingGroup
      ComparisonOperator: GreaterThanThreshold
  
  CPUAlarmLow:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: Scale down if CPU < 30%
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 30
      AlarmActions:
        - !Ref ScaleDownPolicy
      Dimensions:
        - Name: AutoScalingGroupName
          Value: !Ref AutoScalingGroup
      ComparisonOperator: LessThanThreshold

Outputs:
  LoadBalancerURL:
    Description: Load Balancer URL
    Value: !Sub 'http://${ApplicationLoadBalancer.DNSName}'
  
  TargetGroupArn:
    Description: Target Group ARN
    Value: !Ref TargetGroup
```

## RDS Database

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: RDS MySQL database with Multi-AZ

Parameters:
  EnvironmentName:
    Type: String
    Default: Production
  
  DBInstanceClass:
    Type: String
    Default: db.t3.micro
    AllowedValues:
      - db.t3.micro
      - db.t3.small
      - db.t3.medium
      - db.r5.large
  
  DBName:
    Type: String
    Default: mydatabase
  
  DBUsername:
    Type: String
    Default: admin
    NoEcho: true
  
  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 8
  
  AllocatedStorage:
    Type: Number
    Default: 20
    MinValue: 20
    MaxValue: 1000
  
  MultiAZ:
    Type: String
    Default: 'true'
    AllowedValues:
      - 'true'
      - 'false'

Resources:
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Subnet group for RDS
      SubnetIds:
        Fn::Split:
          - ','
          - Fn::ImportValue: !Sub ${EnvironmentName}-PRIV-NETS
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-DBSubnetGroup
  
  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Database Security Group
      VpcId:
        Fn::ImportValue: !Sub ${EnvironmentName}-VPCID
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId:
            Fn::ImportValue: !Sub ${EnvironmentName}-WebServerSG
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-DBSG
  
  DBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBName: !Ref DBName
      Engine: mysql
      EngineVersion: '8.0.35'
      DBInstanceClass: !Ref DBInstanceClass
      AllocatedStorage: !Ref AllocatedStorage
      StorageType: gp3
      StorageEncrypted: true
      MasterUsername: !Ref DBUsername
      MasterUserPassword: !Ref DBPassword
      DBSubnetGroupName: !Ref DBSubnetGroup
      VPCSecurityGroups:
        - !Ref DBSecurityGroup
      MultiAZ: !Ref MultiAZ
      BackupRetentionPeriod: 7
      PreferredBackupWindow: '03:00-04:00'
      PreferredMaintenanceWindow: 'sun:04:00-sun:05:00'
      EnableCloudwatchLogsExports:
        - error
        - general
        - slowquery
      DeletionProtection: true
      Tags:
        - Key: Name
          Value: !Sub ${EnvironmentName}-MySQL
    DeletionPolicy: Snapshot

Outputs:
  DBEndpoint:
    Description: Database endpoint
    Value: !GetAtt DBInstance.Endpoint.Address
    Export:
      Name: !Sub ${EnvironmentName}-DBEndpoint
  
  DBPort:
    Description: Database port
    Value: !GetAtt DBInstance.Endpoint.Port
    Export:
      Name: !Sub ${EnvironmentName}-DBPort
```

## Nested Stacks

**master-stack.yaml:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Master stack

Parameters:
  EnvironmentName:
    Type: String
    Default: Production

Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/mybucket/network.yaml
      Parameters:
        EnvironmentName: !Ref EnvironmentName
      TimeoutInMinutes: 10
  
  SecurityStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: https://s3.amazonaws.com/mybucket/security.yaml
      Parameters:
        EnvironmentName: !Ref EnvironmentName
        VPCId: !GetAtt NetworkStack.Outputs.VPC
      TimeoutInMinutes: 5
  
  ApplicationStack:
    Type: AWS::CloudFormation::Stack
    DependsOn:
      - NetworkStack
      - SecurityStack
    Properties:
      TemplateURL: https://s3.amazonaws.com/mybucket/application.yaml
      Parameters:
        EnvironmentName: !Ref EnvironmentName
        PublicSubnets: !GetAtt NetworkStack.Outputs.PublicSubnets
        PrivateSubnets: !GetAtt NetworkStack.Outputs.PrivateSubnets
        WebServerSG: !GetAtt SecurityStack.Outputs.WebServerSG
      TimeoutInMinutes: 15

Outputs:
  ApplicationURL:
    Description: Application URL
    Value: !GetAtt ApplicationStack.Outputs.LoadBalancerURL
```

## Intrinsic Functions

```yaml
# Ref - Reference parameter or resource
!Ref InstanceType
!Ref EC2Instance

# GetAtt - Get attribute
!GetAtt EC2Instance.PublicIp
!GetAtt LoadBalancer.DNSName

# Sub - Substitute variables
!Sub '${EnvironmentName}-VPC'
!Sub 'http://${LoadBalancer.DNSName}'

# Join - Join list
!Join [',', [subnet-1, subnet-2, subnet-3]]
!Join ['', ['http://', !Ref LoadBalancer]]

# Split - Split string
!Split [',', !ImportValue SubnetList]

# Select - Select from list
!Select [0, !GetAZs '']  # First AZ
!Select [1, !Ref SubnetList]

# FindInMap - Get value from mapping
!FindInMap [RegionMap, !Ref 'AWS::Region', AMI]

# ImportValue - Import from another stack
!ImportValue Production-VPCID

# GetAZs - Get availability zones
!GetAZs ''
!GetAZs 'us-east-1'

# If - Conditional
!If [CreateProdResources, m5.large, t3.micro]

# Equals, And, Or, Not
!Equals [!Ref Environment, 'Production']
!And [!Equals [!Ref Env, Prod], !Equals [!Ref Region, us-east-1]]
!Or [!Equals [!Ref Env, Dev], !Equals [!Ref Env, Test]]
!Not [!Equals [!Ref Env, Production]]

# Base64 - Encode to base64
!Base64 |
  #!/bin/bash
  echo "Hello World"

# Cidr - Generate CIDR blocks
!Cidr [10.0.0.0/16, 6, 8]  # 6 subnets with /24
```

## Change Sets

```bash
# Create change set
aws cloudformation create-change-set \
  --stack-name my-stack \
  --change-set-name my-changes \
  --template-body file://template.yaml \
  --parameters file://parameters.json

# Describe change set
aws cloudformation describe-change-set \
  --stack-name my-stack \
  --change-set-name my-changes

# Execute change set
aws cloudformation execute-change-set \
  --stack-name my-stack \
  --change-set-name my-changes

# Delete change set
aws cloudformation delete-change-set \
  --stack-name my-stack \
  --change-set-name my-changes
```

## Stack Policies

**stack-policy.json:**
```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "Update:*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "Update:Delete",
      "Resource": "LogicalResourceId/ProductionDatabase"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "Update:Replace",
      "Resource": "LogicalResourceId/ProductionDatabase"
    }
  ]
}
```

```bash
# Set stack policy
aws cloudformation set-stack-policy \
  --stack-name my-stack \
  --stack-policy-body file://stack-policy.json

# Get stack policy
aws cloudformation get-stack-policy \
  --stack-name my-stack
```

## Best Practices

### 1. Use Parameters

```yaml
Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, production]
  
  InstanceType:
    Type: String
    Default: t2.micro
  
  DBPassword:
    Type: String
    NoEcho: true
    MinLength: 8
```

### 2. Use Mappings for Static Values

```yaml
Mappings:
  EnvironmentConfig:
    dev:
      InstanceType: t2.micro
      DBClass: db.t3.micro
    production:
      InstanceType: m5.large
      DBClass: db.r5.large

Resources:
  Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !FindInMap [EnvironmentConfig, !Ref Environment, InstanceType]
```

### 3. Use Conditions

```yaml
Conditions:
  CreateProdResources: !Equals [!Ref Environment, production]
  CreateDevResources: !Equals [!Ref Environment, dev]

Resources:
  DevInstance:
    Type: AWS::EC2::Instance
    Condition: CreateDevResources
    Properties:
      InstanceType: t2.micro
  
  ProdInstance:
    Type: AWS::EC2::Instance
    Condition: CreateProdResources
    Properties:
      InstanceType: m5.large
```

### 4. Export/Import Values

```yaml
# Stack 1 - Export
Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub ${EnvironmentName}-VPCID

# Stack 2 - Import
Parameters:
  EnvironmentName:
    Type: String

Resources:
  SecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      VpcId: !ImportValue
        Fn::Sub: ${EnvironmentName}-VPCID
```

### 5. DeletionPolicy

```yaml
Resources:
  Database:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot  # Create snapshot before delete
    Properties:
      # ...
  
  S3Bucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain  # Keep even after stack delete
```

### 6. DependsOn

```yaml
Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    DependsOn: InternetGatewayAttachment
    Properties:
      # ...
  
  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      # ...
```

### 7. Use Stack Tags

```bash
aws cloudformation create-stack \
  --stack-name my-stack \
  --template-body file://template.yaml \
  --tags Key=Environment,Value=Production \
         Key=Project,Value=MyApp \
         Key=CostCenter,Value=Engineering
```

## CloudFormation CLI

```bash
# Validate template
aws cloudformation validate-template \
  --template-body file://template.yaml

# Estimate cost
aws cloudformation estimate-template-cost \
  --template-body file://template.yaml \
  --parameters file://parameters.json

# List stack resources
aws cloudformation list-stack-resources \
  --stack-name my-stack

# Describe stack events
aws cloudformation describe-stack-events \
  --stack-name my-stack

# Get template
aws cloudformation get-template \
  --stack-name my-stack

# Cancel update
aws cloudformation cancel-update-stack \
  --stack-name my-stack

# Continue update rollback
aws cloudformation continue-update-rollback \
  --stack-name my-stack

# Detect stack drift
aws cloudformation detect-stack-drift \
  --stack-name my-stack

# Describe drift
aws cloudformation describe-stack-resource-drifts \
  --stack-name my-stack
```

## Troubleshooting

### Common Issues

```bash
# Stack stuck in UPDATE_ROLLBACK_FAILED
aws cloudformation continue-update-rollback \
  --stack-name my-stack \
  --resources-to-skip EC2Instance

# Stack stuck in DELETE_FAILED
# 1. Check Events
aws cloudformation describe-stack-events \
  --stack-name my-stack | grep FAILED

# 2. Manually delete resource
# 3. Retry delete
aws cloudformation delete-stack --stack-name my-stack

# View CloudFormation logs
aws cloudformation describe-stack-events \
  --stack-name my-stack \
  --max-items 50

# Check drift
aws cloudformation detect-stack-drift --stack-name my-stack
aws cloudformation describe-stack-resource-drifts --stack-name my-stack
```

**Tổng kết:**
- Template-based IaC
- JSON/YAML syntax
- Intrinsic functions
- Nested stacks
- Change sets
- Stack policies
- Drift detection

Master CloudFormation for AWS infrastructure automation!
