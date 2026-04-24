This README.md documentation focuses on the Compute and Storage layer. it describes the automated deployment of the Ubuntu Application Server, the self-generating Key Pair, and the auto-mounting EBS volume logic using CloudFormation.
------------------------------
## 🖥️ AWS EC2 & Storage Automation (IaC)
This repository contains the CloudFormation template to deploy a pre-configured Ubuntu 24.04 server. It automates the identity, security, and storage scaling required for a production-ready node.
## 🚀 Key Features

* OS: Ubuntu 24.04 LTS (Noble Numbat).
* Security: Automated Security Group creation allowing SSH (22) and HTTP (80).
* Identity: Self-generating RSA Key Pair managed via AWS Systems Manager (SSM).
* Storage Expansion: 10GB EBS Volume (gp3) attached as a secondary drive.
* Bootstrap Script: Automated UserData script to format and mount the EBS volume to /data_store on boot.

------------------------------
## 📄 Compute & Storage YAML Template
Save this code as ec2_deploy.yaml. Note that this template is designed to launch in your Default VPC for maximum simplicity.

AWSTemplateFormatVersion: '2010-09-09'Description: 'CloudFormation template for Ubuntu 24.04, KeyPair, and Auto-Mounted EBS.'
Parameters:
  LatestUbuntuAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/canonical/ubuntu/server/24.04/stable/current/amd64/hvm/ebs-gp3/ami-id'
Resources:
```
  # 1. Automated Key Pair Generation
  PrimaryAdminKey:
    Type: 'AWS::EC2::KeyPair'
    Properties:
      KeyName: PrimaryAdminKey
      Tags: [{Key: Name, Value: PrimaryAdminKey}]

  # 2. Security Group Configuration
  CloudGatewaySG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH and HTTP access
      SecurityGroupIngress:
        - {IpProtocol: tcp, FromPort: 22, ToPort: 22, CidrIp: 0.0.0.0/0}
        - {IpProtocol: tcp, FromPort: 80, ToPort: 80, CidrIp: 0.0.0.0/0}
      Tags: [{Key: Name, Value: CloudGatewaySG}]

  # 3. EC2 Instance Definition
  MainAppServer:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      ImageId: !Ref LatestUbuntuAmiId
      KeyName: !Ref PrimaryAdminKey
      SecurityGroupIds: [!Ref CloudGatewaySG]
      Tags: [{Key: Name, Value: MainAppServer_Node01}]
      UserData: 
        Fn::Base64: |
          #!/bin/bash
          # Wait for volume, format, and mount
          while [ ! -b /dev/xvdh ]; do sleep 5; done
          mkfs -t ext4 /dev/xvdh
          mkdir /data_store
          mount /dev/xvdh /data_store
          echo "/dev/xvdh /data_store ext4 defaults,nofail 0 2" >> /etc/fstab
  # 4. 10GB External EBS Volume
  ExternalDataDisk:
    Type: AWS::EC2::Volume
    Properties:
      Size: 10
      VolumeType: gp3
      AvailabilityZone: !GetAtt MainAppServer.AvailabilityZone
      Tags: [{Key: Name, Value: AppServer_DataDisk}]

  # 5. Volume Attachment
  DiskAttachment:
    Type: AWS::EC2::VolumeAttachment
    Properties:
      InstanceId: !Ref MainAppServer
      VolumeId: !Ref ExternalDataDisk
      Device: /dev/sdh
Outputs:
  InstancePublicIP:
    Description: "Public IP of the server"
    Value: !GetAtt MainAppServer.PublicIp
  KeyPairID:
    Description: "ID of the key pair stored in SSM"
    Value: !Ref PrimaryAdminKey
```

   
   


