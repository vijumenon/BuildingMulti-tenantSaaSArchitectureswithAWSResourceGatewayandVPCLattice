# Secure Customer Resource Access in Multi-tenant SaaS with VPC Lattice

This repo is to support the blog post at -# Secure Customer Resource Access in Multi-tenant SaaS with VPC Lattice

This repo is to support the blog post at https://amazon.awsapps.com/workdocs-amazon/index.html#/document/fc04cd32d9a20c623cedb051bde7ef2991e7ddcd88f34e06513ccf9bca729dd7 . Below are the steps to deploy the solution in your own AWS environment.

## Prerequisites

Following resources are required to walk through the steps:

* Preferably 3 separate AWS accounts to simulate one SaaS provider and two customers. For testing purposes you can also use the same account for SaaS provider and customers and skip the resource sharing parts in the instructions. We use 3 separate AWS accounts in the steps to keep it as close as possible to real world scenarios. 
* One VPC per account, with an EC2 instance in the SaaS provider VPC to simulate the Service, and a resource in each customer VPC (such as a database or an EC2 instance listening on a TCP port) to simulate the customer resource. In this blog post we used a sample TCP app listening on TCP port 1234. We will be using this CloudFormation template to deploy the base infrastructure.
* AWS Command Line Interface (AWS CLI): You need AWS CLI installed and configured on the workstation from where you are going to try the steps mentioned below.
* Access to credentials to be configured in AWS CLI, which should have the required IAM permissions to spin up and modify the resources mentioned in this post for each of the 3 accounts mentioned above.
* This blog post assumes that us-east-1 Region is used and your AWS CLI default Region is us-east-1. If us-east-1 is not the default Region, please mentioned the Regions explicitly while executing AWS CLI commands using --region us-east-1 switch.

### Considerations:

* The compute platform doesn't matter as long as it sits in the VPC. We will be using an EC2 instance to showcase connectivity between service and resource. 
* For zonal consistency between SaaS provider VPC and customer VPCs make sure that you are using all AZs in the SaaS provider side VPC for service network endpoints. 
* In the SaaS provider side VPC, we recommend a dedicated subnet for the service network endpoints in every AZ, each with one IP per resource you will expect. For example if your tenants all share one resource each and you expect no more than 100 tenants to share resources with you, /25 subnets would be sufficient to host the 100 IP addresses.

## Implementation Guide for A - Dedicated Service Network per Customer

### High level steps

#### Customer side:

1. In the customer account, create a Resource Gateway in the VPC where the resource in question is located. While creating the Resource Gateway, choose at-least two subnets across two AZs and the Security Group with required rules in it.
2. Create a Resource Configuration using the Resource Gateway created in previous step. The Resource Configuration should be created using the TCP port the resource is listening on and in resource-configuration-definition, type should be chosen as ipResource and should point to the IP address of the TCP server.
3. Create a Service Network and associate the Resource Configuration created in previous step to this Service Network. 
3. Create a Resource Share for this Service Network and share it with the SaaS provider account.
4. Repeat these steps for other customer accounts.

#### SaaS provider side:

1. In the SaaS provider account, accept the Resource Share Invitation from  customer accounts.
2. Create a service network endpoint for the service network of each customer in the SaaS provider VPC. Make sure to choose all the AZs so that AZ mismatch between customer and SaaS provider can be avoided. Alternatively you can align AZs between Resource Gateways in customer VPCs and service network endpoint in the SaaS provider VPC.
4. Obtain the DNS Names assigned by the system for each customer resource and test connectivity.
5. Optionally, create and associate a Private Hosted Zone with the SaaS provider VPC and create CNAME records to point the system-generated DNS names to more user friendly domain names.

### Detailed steps

#### Setting up the customer side

Let's begin with setting up the customer side by creating a Resource Gateway, then creating a Resource Configuration using this newly created Resource Gateway and resource/s to be exposed to the SaaS provider. First download this CloudFormation template ([RGBlog.yaml](https://gitlab.aws.dev/vijaym/LatticeResourceGWBlog/-/raw/main/RGBlog.yaml?ref_type=heads&inline=false)) which we will use to deploy the base infrastructure in all accounts.


##### Customer 1:

1. Switch Roles or use credentials to log on to the 1st customer account from your terminal.
2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name consumer1vpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Service-Consumer1 \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id such as one shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:001182731686:stack/consumer1vpc/28effa40-06d2-11f0-adb6-122d7d2f3903"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name consumer1vpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk:

```bash
aws cloudformation list-stack-resources --stack-name consumer1vpc > consumer1vpc.json
```

The file created in this step will have content similar to the one shown in the snippet below (we have removed the resources that are not relevant to the instructions):

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-0535f65e2e5db2c36",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-06-13T08:21:42.255000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-0e729e42439b39c00",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-06-13T08:22:12.909000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-0c7251d3cd3580f4c",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-06-13T08:21:32.822000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-07ed64c78bf050e3e",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:21:58.023000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-0bf45cfa188d0190d",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:21:58.471000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
```

4. Create a Resource Gateway using the above information:

```bash
aws vpc-lattice create-resource-gateway --name consumer1gateway --vpc-identifier vpc-0c7251d3cd3580f4c --subnet-ids subnet-0bf45cfa188d0190d subnet-07ed64c78bf050e3e --security-group-ids sg-0535f65e2e5db2c36 --ip-address-type IPV4
```

The output will look like this (make a note of the Resource Gateway arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourcegateway/rgw-06745d5fa17e71678",
    "id": "rgw-06745d5fa17e71678",
    "ipAddressType": "IPV4",
    "name": "consumer1gateway",
    "securityGroupIds": [
        "sg-0535f65e2e5db2c36"
    ],
    "status": "ACTIVE",
    "subnetIds": [
        "subnet-0bf45cfa188d0190d",
        "subnet-07ed64c78bf050e3e"
    ],
    "vpcIdentifier": "vpc-0c7251d3cd3580f4c"
}
```

5. Create a Resource Configuration using the Resource Gateway:

First, retrieve the private IP address of the TCP server:

```bash
aws ec2 describe-instances --instance-ids i-0e729e42439b39c00 --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
```

Output (make a note of it):
```
10.0.1.14
```

Then create the Resource Configuration:

```bash
aws vpc-lattice create-resource-configuration --name consumer1resourceconfig --resource-gateway-identifier rgw-06745d5fa17e71678  --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=10.0.1.14}'
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "allowAssociationToShareableServiceNetwork": true,
    "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-03eb9d30003e44cbc",
    "id": "rcfg-03eb9d30003e44cbc",
    "name": "consumer1resourceconfig",
    "portRanges": [
        "1234"
    ],
    "protocol": "TCP",
    "resourceConfigurationDefinition": {
        "ipResource": {
            "ipAddress": "10.0.1.14"
        }
    },
    "resourceGatewayId": "rgw-06745d5fa17e71678",
    "status": "ACTIVE",
    "type": "SINGLE"
}
```

6. Create a Service Network with '--sharing-config' flag as 'enabled=true'

```bash
aws vpc-lattice create-service-network --name cx1-service-network --sharing-config enabled=true
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetworkresourceassociation/snra-08ab290695bf1e2f5",
    "createdBy": "661497052113",
    "id": "snra-08ab290695bf1e2f5",
    "status": "CREATE_IN_PROGRESS"
}
```
7. Associate the Resource Configuration from step 5 with Service Network from step 6.

```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier rcfg-02068184d728a2883 --service-network-identifier sn-0a414e0c612df5a26
```

Output will be similar to this

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetworkresourceassociation/snra-06dd45332f4d40737",
    "createdBy": "001182731686",
    "id": "snra-06dd45332f4d40737",
    "status": "CREATE_IN_PROGRESS"
}
```


8. Create a Resource Share for SaaS provider account by providing SaaS provider account id as the value of '--principal' switch.

```bash
aws ram create-resource-share --name consumer1sn --principals 214517638895
```

The output will be similar to this (make a note of the arn):

```json
{
    "resourceShare": {
        "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21",
        "name": "consumer1sn",
        "owningAccountId": "661497052113",
        "allowExternalPrincipals": true,
        "status": "ACTIVE",
        "tags": [],
        "creationTime": "2025-06-14T09:24:35.985000+08:00",
        "lastUpdatedTime": "2025-06-14T09:24:35.985000+08:00"
    }
}
```

9. Associate the Service Network arn from step 6 with the resource share arn from step 8.

```bash
aws ram associate-resource-share --resource-share arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21 --resource-arns arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b
```

The output will be similar to this (make a note of the arn):

```json
{
    "resourceShareAssociations": [
        {
            "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21",
            "associatedEntity": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b",
            "associationType": "RESOURCE",
            "status": "ASSOCIATING",
            "external": false
        }
    ]
}
```

##### Customer 2 Setup

1. Switch Roles or use credentials to log on to the second customer account from your terminal.

2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name consumer2vpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Service-Consumer2 \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id such as one shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:001182731686:stack/consumer2vpc/28effa40-06d2-11f0-adb6-122d7d2f3903"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name consumer2vpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk:

```bash
aws cloudformation list-stack-resources --stack-name consumer2vpc > consumer2vpc.json
```

The file created in this step will have content similar to the one shown in the snippet below (we have removed the resources that are not relevant to the instructions):

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-09e324e0e2fbab9f6",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-06-13T08:23:02.564000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-0b8a4392dc722b7a7",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-06-13T08:23:32.126000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-074bcdd680fdae720",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-06-13T08:22:52.625000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-08b91245a53c4d631",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:23:17.215000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-01a5eba50b4ba6dde",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:23:17.215000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
```

4. Create a Resource Gateway using the above information:

```bash
aws vpc-lattice create-resource-gateway --name consumer2gateway --vpc-identifier vpc-074bcdd680fdae720 --subnet-ids subnet-08b91245a53c4d631 subnet-01a5eba50b4ba6dde --security-group-ids sg-09e324e0e2fbab9f6 --ip-address-type IPV4
```

The output will look like this (make a note of the Resource Gateway arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourcegateway/rgw-01da2681a040c1d94",
    "id": "rgw-01da2681a040c1d94",
    "ipAddressType": "IPV4",
    "name": "consumer2gateway",
    "securityGroupIds": [
        "sg-09e324e0e2fbab9f6"
    ],
    "status": "ACTIVE",
    "subnetIds": [
        "subnet-08b91245a53c4d631",
        "subnet-01a5eba50b4ba6dde"
    ],
    "vpcIdentifier": "vpc-074bcdd680fdae720"
}
```

5. Create a Resource Configuration using the Resource Gateway:

First, retrieve the private IP address of the TCP server:

```bash
aws ec2 describe-instances --instance-ids i-0b8a4392dc722b7a7 --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
```

Output (make a note of it):
```
10.0.1.157
```

Then create the Resource Configuration:

```bash
aws vpc-lattice create-resource-configuration --name consumer2resourceconfig --resource-gateway-identifier rgw-01da2681a040c1d94 --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=10.0.1.157}'
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "allowAssociationToShareableServiceNetwork": true,
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-02068184d728a2883",
    "id": "rcfg-02068184d728a2883",
    "name": "consumer2resourceconfig",
    "portRanges": [
        "1234"
    ],
    "protocol": "TCP",
    "resourceConfigurationDefinition": {
        "ipResource": {
            "ipAddress": "10.0.1.157"
        }
    },
    "resourceGatewayId": "rgw-01da2681a040c1d94",
    "status": "ACTIVE",
    "type": "SINGLE"
}
```

6. Create a Service Network with '--sharing-config' flag as 'enabled=true'

```bash
aws vpc-lattice create-service-network --name cx2-service-network --sharing-config enabled=true
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26",
    "authType": "NONE",
    "id": "sn-0a414e0c612df5a26",
    "name": "cx2-service-network",
    "sharingConfig": {
        "enabled": true
    }
}
```

7. Associate the Resource Configuration from step 5 with Service Network from step 6.

```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier rcfg-02068184d728a2883 --service-network-identifier sn-0a414e0c612df5a26
```

Output will be similar to this

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetworkresourceassociation/snra-06dd45332f4d40737",
    "createdBy": "001182731686",
    "id": "snra-06dd45332f4d40737",
    "status": "CREATE_IN_PROGRESS"
}
```


8. Create a Resource Share for SaaS provider account by providing SaaS provider account id as the value of '--principal' switch.

```bash
aws ram create-resource-share --name consumer2sn --principals 214517638895
```

The output will be similar to this (make a note of the arn):

```json
{
    "resourceShare": {
        "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42",
        "name": "consumer2sn",
        "owningAccountId": "001182731686",
        "allowExternalPrincipals": true,
        "status": "ACTIVE",
        "tags": [],
        "creationTime": "2025-06-14T09:29:46.558000+08:00",
        "lastUpdatedTime": "2025-06-14T09:29:46.558000+08:00"
    }
}
```

9. Associate the Service Network arn from step 6 with the resource share arn from step 8.

```bash
aws ram associate-resource-share --resource-share arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42 --resource-arns arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26
```

The output will be similar to this (make a note of the arn):

```json
{
    "resourceShareAssociations": [
        {
            "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42",
            "associatedEntity": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26",
            "associationType": "RESOURCE",
            "status": "ASSOCIATING",
            "external": false
        }
    ]
}
```

# Setting up the SaaS Provider Side

1. Switch roles or use credentials to log on to the SaaS provider account from your terminal.

2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name providervpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Provider \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id such as the one shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:214517638895:stack/providervpc/6e2e4950-06e5-11f0-8245-1207e77f631b"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name <name of your stack from above step> | grep StackStatus
```

Example:
```bash
aws cloudformation describe-stacks --stack-name providervpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk. We will be using these IDs throughout this blog post.

```bash
aws cloudformation list-stack-resources --stack-name providervpc > providervpc.json 
```

The file created in this step will have content similar to the one shown in the snippet below. We have removed the resources not being used in the instructions:

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-0fa7fce98345c9604",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-06-13T08:24:19.336000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-0693552b44c4e995f",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-06-13T08:24:50.228000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-069a64174df6950b7",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-06-13T08:24:09.488000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-0e164f16b26e7a329",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:24:34.964000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-0730b2578a2456c07",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:24:37.084000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
```

4. Obtain the Resource Share Invitation arns shared by the two customer accounts. We need access to the Resource Shares to retrieve the Resource Configurations that were shared.

```bash
aws ram get-resource-share-invitations
```

Output example below, make note of the resourceShareInvitationArns:

```json
{
    "resourceShareInvitations": [
        {
            "resourceShareInvitationArn": "arn:aws:ram:us-east-1:661497052113:resource-share-invitation/0124cfbf-ebda-4bbf-bcb8-b3ac487630c1",
            "resourceShareName": "consumer1sn",
            "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21",
            "senderAccountId": "661497052113",
            "receiverAccountId": "214517638895",
            "invitationTimestamp": "2025-06-14T09:24:36.979000+08:00",
            "status": "PENDING"
        },
        {
            "resourceShareInvitationArn": "arn:aws:ram:us-east-1:001182731686:resource-share-invitation/5ff2ac29-0bff-44f0-ac91-92a2cf656218",
            "resourceShareName": "consumer2sn",
            "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42",
            "senderAccountId": "001182731686",
            "receiverAccountId": "214517638895",
            "invitationTimestamp": "2025-06-14T09:29:46.760000+08:00",
            "status": "PENDING"
        }
    ]
}
```

5. Accept both Resource Share invitations using the following command one at a time, so that we can use those Resource Configuration with our service network in subsequent steps.

```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn <resourceShareInvitationArn from step 3>
```

Example:
```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn arn:aws:ram:us-east-1:661497052113:resource-share-invitation/0124cfbf-ebda-4bbf-bcb8-b3ac487630c1
```

Output:
```json
{
    "resourceShareInvitation": {
        "resourceShareInvitationArn": "arn:aws:ram:us-east-1:661497052113:resource-share-invitation/0124cfbf-ebda-4bbf-bcb8-b3ac487630c1",
        "resourceShareName": "consumer1sn",
        "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21",
        "senderAccountId": "661497052113",
        "receiverAccountId": "214517638895",
        "invitationTimestamp": "2025-06-14T09:33:56.662000+08:00",
        "status": "ACCEPTED"
    }
}
```

```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn arn:aws:ram:us-east-1:001182731686:resource-share-invitation/5ff2ac29-0bff-44f0-ac91-92a2cf656218
```

Output:
```json
{
    "resourceShareInvitation": {
        "resourceShareInvitationArn": "arn:aws:ram:us-east-1:001182731686:resource-share-invitation/5ff2ac29-0bff-44f0-ac91-92a2cf656218",
        "resourceShareName": "consumer2sn",
        "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42",
        "senderAccountId": "001182731686",
        "receiverAccountId": "214517638895",
        "invitationTimestamp": "2025-06-14T09:34:14.828000+08:00",
        "status": "ACCEPTED"
    }
}
```

6. List the shared Service Networks for customer 1 and 2 and make not of the respective ARNs.

```bash
aws vpc-lattice list-service-networks
```

Output example below, make note of the ARNs:

```json
{
    "items": [
        {
            "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26",
            "createdAt": "2025-06-14T01:27:51.920000+00:00",
            "id": "sn-0a414e0c612df5a26",
            "lastUpdatedAt": "2025-06-14T01:27:51.920000+00:00",
            "name": "cx2-service-network",
            "numberOfAssociatedResourceConfigurations": 1,
            "numberOfAssociatedServices": 0,
            "numberOfAssociatedVPCs": 0
        },
        {
            "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b",
            "createdAt": "2025-06-14T01:23:08.249000+00:00",
            "id": "sn-0f7a8ceee967ecf4b",
            "lastUpdatedAt": "2025-06-14T01:23:08.249000+00:00",
            "name": "cx1-service-network",
            "numberOfAssociatedResourceConfigurations": 1,
            "numberOfAssociatedServices": 0,
            "numberOfAssociatedVPCs": 0
        }
    ]
}
```

7. Create Service Network Endpoints for customer 1 in provider VPC using the Service Network ARNs from the previous steps and other parameters form step 3.

```bash
aws ec2 create-vpc-endpoint --vpc-endpoint-type ServiceNetwork --vpc-id vpc-069a64174df6950b7 --subnet-ids subnet-0e164f16b26e7a329 subnet-0730b2578a2456c07  --service-network-arn arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b --security-group-ids sg-0fa7fce98345c9604
```

Output will be similar to the one below. Make note of the "VpcEndpointId"

```json
{
    "VpcEndpoint": {
        "VpcEndpointId": "vpce-0057aabf75896925f",
        "VpcEndpointType": "ServiceNetwork",
        "VpcId": "vpc-069a64174df6950b7",
        "State": "Pending",
        "SubnetIds": [
            "subnet-0e164f16b26e7a329",
            "subnet-0730b2578a2456c07"
        ],
        "Groups": [
            {
                "GroupId": "sg-0fa7fce98345c9604",
                "GroupName": "onesnpercx-LocalTrafficSecurityGroup-GLcJ2TGuYPt3"
            }
        ],
        "IpAddressType": "IPV4",
        "PrivateDnsEnabled": false,
        "CreationTimestamp": "2025-06-14T01:52:18.467000+00:00",
        "Tags": [],
        "OwnerId": "214517638895",
        "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b"
    }
}
```

8. Create Service Network Endpoints for customer 2 in provider VPC using the Service Network ARNs from the previous steps and other parameters form step 3.

```bash
aws ec2 create-vpc-endpoint --vpc-endpoint-type ServiceNetwork --vpc-id vpc-069a64174df6950b7 --subnet-ids subnet-0e164f16b26e7a329 subnet-0730b2578a2456c07  --service-network-arn arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26 --security-group-ids sg-0fa7fce98345c9604
```

Output will be similar to the one below. Make note of the "VpcEndpointId"

```json
{
    "VpcEndpoint": {
        "VpcEndpointId": "vpce-0f067bfe89cb48bb2",
        "VpcEndpointType": "ServiceNetwork",
        "VpcId": "vpc-069a64174df6950b7",
        "State": "Pending",
        "SubnetIds": [
            "subnet-0e164f16b26e7a329",
            "subnet-0730b2578a2456c07"
        ],
        "Groups": [
            {
                "GroupId": "sg-0fa7fce98345c9604",
                "GroupName": "onesnpercx-LocalTrafficSecurityGroup-GLcJ2TGuYPt3"
            }
        ],
        "IpAddressType": "IPV4",
        "PrivateDnsEnabled": false,
        "CreationTimestamp": "2025-06-14T01:53:26.456000+00:00",
        "Tags": [],
        "OwnerId": "214517638895",
        "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26"
    }
}
```

## Testing Connectivity

Now that the service network endpoints for each customer is created in SaaS provider VPC, it's time to test the solution end to end.

1. Find out the FQDN for the customer side resources through the service network endpoint details in the SaaS provider account. Start with describing the VPC endpoints using the following command in the SaaS provider account. Make sure your terminal still has the credentials for the SaaS provider account active.

```bash
aws ec2 describe-vpc-endpoint-associations
```

The example output below shows the DNS names assigned to each resource. Make note of these DNS names and their corresponding customer service networks, which can be found in "DnsName" under "DnsEntry" coresponding to each "ServiceNetworkName". This mapping helps identify which DNS name belongs to which customer's resource.

``

Output will be similar to the one below. Make note of the value of "DnsName" under "DnsEntry" coresponding to each "ServiceNetworkName".

```json
aws ec2 describe-vpc-endpoint-associations --vpc-endpoint-ids vpce-0057aabf75896925f vpce-0f067bfe89cb48bb2
{
    "VpcEndpointAssociations": [
        {
            "Id": "vpce-rsc-asc-03f2f67b4d6a5e7d5",
            "VpcEndpointId": "vpce-0057aabf75896925f",
            "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b",
            "ServiceNetworkName": "cx1-service-network",
            "AssociatedResourceAccessibility": "Accessible",
            "DnsEntry": {
                "DnsName": "vpce-0057aabf75896925f-snra-08ab290695bf1e2f5.rcfg-03eb9d30003e44cbc.4232ccc.vpc-lattice-rsc.us-east-1.on.aws",
                "HostedZoneId": "Z08285483B41F6ASEM5U1"
            },
            "AssociatedResourceArn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-03eb9d30003e44cbc"
        },
        {
            "Id": "vpce-rsc-asc-078d3d60c3fbba6b8",
            "VpcEndpointId": "vpce-0f067bfe89cb48bb2",
            "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26",
            "ServiceNetworkName": "cx2-service-network",
            "AssociatedResourceAccessibility": "Accessible",
            "DnsEntry": {
                "DnsName": "vpce-0f067bfe89cb48bb2-snra-06dd45332f4d40737.rcfg-02068184d728a2883.4232ccc.vpc-lattice-rsc.us-east-1.on.aws",
                "HostedZoneId": "Z08285483B41F6ASEM5U1"
            },
            "AssociatedResourceArn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-02068184d728a2883"
        }
    ]
}
```

2. Log on to the EC2 instance in the SaaS provider VPC (for example with Session Manager) and initiate traffic using the DNS Name from the previous step. You should get a successful connection as in the example below. If you remember, we are using a sample application listening on TCP 1234 to simulate a resource in customer VPC.

```bash
sh-4.2$ telnet vpce-0057aabf75896925f-snra-08ab290695bf1e2f5.rcfg-03eb9d30003e44cbc.4232ccc.vpc-lattice-rsc.us-east-1.on.aws 1234
Trying 10.0.1.210...
Connected to vpce-0057aabf75896925f-snra-08ab290695bf1e2f5.rcfg-03eb9d30003e44cbc.4232ccc.vpc-lattice-rsc.us-east-1.on.aws.
Escape character is '^]'.
Connection successful!
Connection closed by foreign host.
sh-4.2$
sh-4.2$
sh-4.2$
sh-4.2$ telnet vpce-0f067bfe89cb48bb2-snra-06dd45332f4d40737.rcfg-02068184d728a2883.4232ccc.vpc-lattice-rsc.us-east-1.on.aws 1234
Trying 10.0.1.40...
Connected to vpce-0f067bfe89cb48bb2-snra-06dd45332f4d40737.rcfg-02068184d728a2883.4232ccc.vpc-lattice-rsc.us-east-1.on.aws.
Escape character is '^]'.
Connection successful!
Connection closed by foreign host.
sh-4.2$
```

This concludes the implementation steps for one Service Network per customer option. Next section covers the implementation steps for shared Service Network option.

--------------


## Implementation Guide for B - Shared Service Network

### High level steps

#### Customer side:

1. In the customer account, create a Resource Gateway in the VPC where the resource in question is located. While creating the Resource Gateway, choose at-least two subnets across two AZs and the Security Group with required rules in it.
2. Create a Resource Configuration using the Resource Gateway created in previous step. The Resource Configuration should be created using the TCP port the resource is listening on and in resource-configuration-definition, type should be chosen as ipResource and should point to the IP address of the TCP server.
3. Create a Resource Share for this Resource Configuration and share it with the SaaS provider account.
4. Repeat these steps for other customer accounts.

#### SaaS provider side:

1. In the SaaS provider account, accept the Resource Share Invitation from all the customer accounts.
2. Create a service network with --sharing-config enabled=false . By default, sharing config is set to true, which will not allow shared Resource Configuration association with the service network.
3. Create a service network endpoint for the service network in the SaaS provider VPC. Make sure to choose all the AZs so that AZ mismatch between customer and SaaS provider can be avoided. Alternatively you can align AZs between Resource Gateways in customer VPCs and service network endpoint in the SaaS provider VPC.
4. Obtain the DNS Names assigned by the system for each customer resource and test connectivity.
5. Optionally, create and associate a Private Hosted Zone with the SaaS provider VPC and create CNAME records to point the system-generated DNS names to more user.friendly domain names.

### Detailed steps

#### Setting up the customer side

Let's begin with setting up the customer side by creating a Resource Gateway, then creating a Resource Configuration using this newly created Resource Gateway and resource/s to be exposed to the SaaS provider. First download this CloudFormation template ([RGBlog.yaml](https://gitlab.aws.dev/vijaym/LatticeResourceGWBlog/-/raw/main/RGBlog.yaml?ref_type=heads&inline=false)) which we will use to deploy the base infrastructure in all accounts.



##### Customer 1:

1. Switch Roles or use credentials to log on to the 1st customer account from your terminal.
2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name consumer1vpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Service-Consumer1 \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id as shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:001182731686:stack/consumer1vpc/28effa40-06d2-11f0-adb6-122d7d2f3903"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name consumer1vpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk:

```bash
aws cloudformation list-stack-resources --stack-name consumer1vpc > consumer1vpc.json
```

The file created in this step will have content similar to the one shown in the snippet below (we have removed the resources that are not relevant to the instructions):

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-09632e5c873e108cd",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-03-22T04:00:40.052000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-08a56ca3848b640b7",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-03-22T04:01:55.275000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-0b0e03534b5f3f8ef",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-03-22T04:00:30.691000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-036da24d3897feb13",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T04:00:35.826000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-00921cc39600a34ec",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T04:00:35.756000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
    ]
}
```

4. Create a Resource Gateway using the above information:

```bash
aws vpc-lattice create-resource-gateway --name consumer1gateway --vpc-identifier vpc-0b0e03534b5f3f8ef --subnet-ids subnet-036da24d3897feb13 subnet-00921cc39600a34ec --security-group-ids sg-09632e5c873e108cd --ip-address-type IPV4
```

The output will look like this (make a note of the Resource Gateway arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourcegateway/rgw-036d78f254aaa2dcf",
    "id": "rgw-036d78f254aaa2dcf",
    "ipAddressType": "IPV4",
    "name": "consumer1gateway",
    "securityGroupIds": [
        "sg-09632e5c873e108cd"
    ],
    "status": "ACTIVE",
    "subnetIds": [
        "subnet-036da24d3897feb13",
        "subnet-00921cc39600a34ec"
    ],
    "vpcIdentifier": "vpc-0b0e03534b5f3f8ef"
}
```

5. Create a Resource Configuration using the Resource Gateway:

First, retrieve the private IP address of the TCP server:

```bash
aws ec2 describe-instances --instance-ids i-08a56ca3848b640b7 --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
```

Output (make a note of it):
```
10.0.1.121
```

Then create the Resource Configuration:

```bash
aws vpc-lattice create-resource-configuration --no-allow-association-to-shareable-service-network --name consumer1resourceconfig --resource-gateway-identifier rgw-036d78f254aaa2dcf  --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=10.0.1.121}'
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "allowAssociationToShareableServiceNetwork": false,
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad",
    "id": "rcfg-0349a6320d67a56ad",
    "name": "consumer1resourceconfig",
    "portRanges": [
        "1234"
    ],
    "protocol": "TCP",
    "resourceConfigurationDefinition": {
        "ipResource": {
            "ipAddress": "10.0.1.121"
        }
    },
    "resourceGatewayId": "rgw-036d78f254aaa2dcf",
    "status": "ACTIVE",
    "type": "SINGLE"
}
```

6. Create a Resource Share using AWS RAM and share it with the SaaS provider account:

```bash
aws ram create-resource-share --name consumer1rcfg --principals 214517638895
```

Output:
```json
{
    "resourceShare": {
        "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d",
        "name": "consumer1rcfg",
        "owningAccountId": "001182731686",
        "allowExternalPrincipals": true,
        "status": "ACTIVE",
        "tags": [],
        "creationTime": "2025-03-22T05:36:57.300000+00:00",
        "lastUpdatedTime": "2025-03-22T05:36:57.300000+00:00"
    }
}
```

7. Associate the Resource Configuration with the Resource Share:

```bash
aws ram associate-resource-share --resource-share arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d --resource-arns arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad
```

Output:
```json
{
    "resourceShareAssociations": [
        {
            "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d",
            "associatedEntity": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad",
            "associationType": "RESOURCE",
            "status": "ASSOCIATING",
            "external": false
        }
    ]
}
```

##### Customer 2 Setup

1. Switch Roles or use credentials to log on to the second customer account from your terminal.

2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name consumer2vpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Service-Consumer2 \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id as shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:661497052113:stack/consumer2vpc/da5e6c60-06d4-11f0-b7b3-0e7cb26ba367"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name <name of your stack from above step> | grep StackStatus
```

Example:
```bash
aws cloudformation describe-stacks --stack-name consumer2vpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk. We will be using these IDs throughout this blog post.

```bash
aws cloudformation list-stack-resources --stack-name consumer2vpc > consumer2vpc.json 
```

The file created in this step will have content similar to the one shown in the snippet below. We have removed the resources not being used in the instructions:

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-043275505a56c7210",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-03-22T04:19:55.396000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-05b8b6e9ef1d17db8",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-03-22T04:21:10.350000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-0fd531b96013cb156",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-03-22T04:19:46.622000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-0bc5b9aa3bec7eee4",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T04:19:51.546000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-0f04b9643e598a98a",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T04:19:53.172000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
    ]
}
```

4. Create a Resource Gateway using the above information using the below command:

```bash
aws vpc-lattice create-resource-gateway --name consumer2gateway --vpc-identifier <vpc-id from step2.1> --subnet-ids <private-subnet-1 id from step 2.a> <private-subnet-2 id from step 2.a> --security-group-ids <LocalTrafficSecurityGroup id from step 2.a> --ip-address-type IPV4
```

For Example:
```bash
aws vpc-lattice create-resource-gateway --name consumer2gateway --vpc-identifier vpc-0fd531b96013cb156 --subnet-ids subnet-0bc5b9aa3bec7eee4 subnet-0f04b9643e598a98a --security-group-ids sg-043275505a56c7210 --ip-address-type IPV4
```

Output will look like the below example, make a note of Resource Gateway arn and id from the output to be used in next step:
```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourcegateway/rgw-0c2ad17dc18ff5bbf",
    "id": "rgw-0c2ad17dc18ff5bbf",
    "ipAddressType": "IPV4",
    "name": "consumer1gateway",
    "securityGroupIds": [
        "sg-043275505a56c7210"
    ],
    "status": "ACTIVE",
    "subnetIds": [
        "subnet-0bc5b9aa3bec7eee4",
        "subnet-0f04b9643e598a98a"
    ],
    "vpcIdentifier": "vpc-0fd531b96013cb156"
}
```

5. Create Resource Configuration using the Resource Gateway created in step 3 and IP address of the TCP server to be used as resource. For that first we need to find the IP address of the EC2 instance running the TCP server.

    a. Find the IP address of the TCP server using the below command:

    ```bash
    aws ec2 describe-instances --instance-ids <instance-id from step 2.a> --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
    ```

    Example:
    ```bash
    aws ec2 describe-instances --instance-ids i-05b8b6e9ef1d17db8 --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
    ```

    Output (make a note of it):
    ```
    10.0.1.43
    ```

    b. Create Resource Configuration using the command below:

    ```bash
    aws vpc-lattice create-resource-configuration --no-allow-association-to-shareable-service-network --name consumer2resourceconfig --resource-gateway-identifier <resource gateway id from step 3>  --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=<ip address from step 4. above>}'
    ```

    Example:
    ```bash
    aws vpc-lattice create-resource-configuration --no-allow-association-to-shareable-service-network --name consumer2resourceconfig --resource-gateway-identifier rgw-0c2ad17dc18ff5bbf  --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=10.0.1.43}'
    ```

    Output will be similar to one shown below, make a note of arn and id for the Resource Configuration from the output:
    ```json
    {
        "allowAssociationToShareableServiceNetwork": false,
        "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd",
        "id": "rcfg-01ae20d51100803bd",
        "name": "consumer2resourceconfig",
        "portRanges": [
            "1234"
        ],
        "protocol": "TCP",
        "resourceConfigurationDefinition": {
            "ipResource": {
                "ipAddress": "10.0.1.43"
            }
        },
        "resourceGatewayId": "rgw-0c2ad17dc18ff5bbf",
        "status": "ACTIVE",
        "type": "SINGLE"
    }
    ```

6. Create a Resource share using AWS RAM and share the Resource Configuration with the SaaS provider account.

```bash
aws ram create-resource-share --name <resource-share-name> --principals <provider-account-id>
```

Example:
```bash
aws ram create-resource-share --name consumer2rcfg --principals 214517638895
```

Output:
```json
{
    "resourceShare": {
        "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185",
        "name": "consumer2rcfg",
        "owningAccountId": "661497052113",
        "allowExternalPrincipals": true,
        "status": "ACTIVE",
        "tags": [],
        "creationTime": "2025-03-22T06:04:56.208000+00:00",
        "lastUpdatedTime": "2025-03-22T06:04:56.208000+00:00"
    }
}
```

7. Associate Resource Configuration with the Resource Share created in above step.

```bash
aws ram associate-resource-share --resource-share <resource share arn from step 5 above> --resource-arns <resource configuration arn from step 4.b>
```

Example:
```bash
aws ram associate-resource-share --resource-share arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185 --resource-arns arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd 
```

Output:
```json
{
    "resourceShareAssociations": [
        {
            "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185",
            "associatedEntity": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd",
            "associationType": "RESOURCE",
            "status": "ASSOCIATING",
            "external": false
        }
    ]
}
```

Now that we have setup the customer environments successfully, let's move on to setting up the SaaS provider side of the solution.

# Setting up the SaaS Provider Side

1. Switch roles or use credentials to log on to the SaaS provider account from your terminal.

2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name providervpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Provider \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id as shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:214517638895:stack/providervpc/6e2e4950-06e5-11f0-8245-1207e77f631b"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name <name of your stack from above step> | grep StackStatus
```

Example:
```bash
aws cloudformation describe-stacks --stack-name providervpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk. We will be using these IDs throughout this blog post.

```bash
aws cloudformation list-stack-resources --stack-name providervpc > providervpc.json 
```

The file created in this step will have content similar to the one shown in the snippet below. We have removed the resources not being used in the instructions:

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-0f6fac5a45e5f093a",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-03-22T06:18:36.825000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-007165e10c4729cc0",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-03-22T06:18:27.566000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-00d8da6295058fced",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T06:18:32.674000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-00d60dc893a32243d",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T06:18:32.632000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
    ]
}
```

4. Obtain the Resource Share Invitation arns shared by the two customer accounts. We need access to the Resource Shares to retrieve the Resource Configurations that were shared.

```bash
aws ram get-resource-share-invitations
```

Output example below, make note of the resourceShareInvitationArns:

```json
{
    "resourceShareInvitations": [
        {
            "resourceShareInvitationArn": "arn:aws:ram:us-east-1:001182731686:resource-share-invitation/1bbc4cb5-b7a3-4153-a2ce-9a3b2ba962da",
            "resourceShareName": "consumer1rcfg",
            "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d",
            "senderAccountId": "001182731686",
            "receiverAccountId": "214517638895",
            "invitationTimestamp": "2025-03-22T05:36:57.631000+00:00",
            "status": "PENDING"
        },
        {
            "resourceShareInvitationArn": "arn:aws:ram:us-east-1:661497052113:resource-share-invitation/87585b25-f486-4b42-9bc3-43b251d06ead",
            "resourceShareName": "consumer2rcfg",
            "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185",
            "senderAccountId": "661497052113",
            "receiverAccountId": "214517638895",
            "invitationTimestamp": "2025-03-22T06:04:56.502000+00:00",
            "status": "PENDING"
        }
    ]
}
```

5. Accept both Resource Share invitations using the following command one at a time, so that we can use those Resource Configuration with our service network in subsequent steps.

```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn <resourceShareInvitationArn from step 3>
```

Example:
```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn arn:aws:ram:us-east-1:001182731686:resource-share-invitation/1bbc4cb5-b7a3-4153-a2ce-9a3b2ba962da
```

Output:
```json
{
    "resourceShareInvitation": {
        "resourceShareInvitationArn": "arn:aws:ram:us-east-1:001182731686:resource-share-invitation/1bbc4cb5-b7a3-4153-a2ce-9a3b2ba962da",
        "resourceShareName": "consumer1rcfg",
        "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d",
        "senderAccountId": "001182731686",
        "receiverAccountId": "214517638895",
        "invitationTimestamp": "2025-03-22T06:36:25.056000+00:00",
        "status": "ACCEPTED"
    }
}
```

```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn arn:aws:ram:us-east-1:661497052113:resource-share-invitation/87585b25-f486-4b42-9bc3-43b251d06ead
```

Output:
```json
{
    "resourceShareInvitation": {
        "resourceShareInvitationArn": "arn:aws:ram:us-east-1:661497052113:resource-share-invitation/87585b25-f486-4b42-9bc3-43b251d06ead",
        "resourceShareName": "consumer2rcfg",
        "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185",
        "senderAccountId": "661497052113",
        "receiverAccountId": "214517638895",
        "invitationTimestamp": "2025-03-22T06:37:39.935000+00:00",
        "status": "ACCEPTED"
    }
}
```

6. Obtain the Resource Configuration details to be used in the service network using the below command.

```bash
aws vpc-lattice list-resource-configurations
```

From the output, make note of the id of each Resource Configuration. For example:

```json
{
    "items": [
        {
            "amazonManaged": false,
            "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd",
            "createdAt": "2025-03-22T06:03:47.880000+00:00",
            "id": "rcfg-01ae20d51100803bd",
            "lastUpdatedAt": "2025-03-22T06:03:47.880000+00:00",
            "name": "consumer1resourceconfig",
            "resourceGatewayId": "rgw-0c2ad17dc18ff5bbf",
            "status": "ACTIVE",
            "type": "SINGLE"
        },
        {
            "amazonManaged": false,
            "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad",
            "createdAt": "2025-03-22T05:27:33.546000+00:00",
            "id": "rcfg-0349a6320d67a56ad",
            "lastUpdatedAt": "2025-03-22T05:27:33.546000+00:00",
            "name": "consumer2resourceconfig",
            "resourceGatewayId": "rgw-036d78f254aaa2dcf",
            "status": "ACTIVE",
            "type": "SINGLE"
        }
    ]
}
```

7. Create a service network using the following command.

```bash
aws vpc-lattice create-service-network --name providersne --sharing-config enabled=false
```

From the output of above command, make note of the service network arn and id, for example:
```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7",
    "authType": "NONE",
    "id": "sn-06ce56215f97909e7",
    "name": "providersne",
    "sharingConfig": {
        "enabled": false
    }
}
```

8. Associate each of the Resource Configurations with the newly created service network using the following command.

```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier <Resource config id from step 5> --service-network-identifier <Service Network id from step 6>
```

Example:
```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier rcfg-01ae20d51100803bd  --service-network-identifier sn-06ce56215f97909e7 
```

Output:
```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetworkresourceassociation/snra-0eae7f31d03c50d72",
    "createdBy": "214517638895",
    "id": "snra-0eae7f31d03c50d72",
    "status": "CREATE_IN_PROGRESS"
}
```

```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier rcfg-0349a6320d67a56ad --service-network-identifier sn-06ce56215f97909e7 
```

Output:
```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetworkresourceassociation/snra-012648bc4e8ad80dc",
    "createdBy": "214517638895",
    "id": "snra-012648bc4e8ad80dc",
    "status": "CREATE_IN_PROGRESS"
}
```

9. Create service network endpoints in the SaaS provider VPC so that our SaaS application in that VPC can access the customer resources.

```bash
aws ec2 create-vpc-endpoint --vpc-endpoint-type ServiceNetwork --vpc-id <vpc-id from step 2.a> --subnet-ids <private subnet 1 id from step 2.a> <private subnet 2 id from step 2.a> --security-group-ids <LocalTrafficSecurityGroup id from step 2.a> --service-network-arn <service network arn from step 6>
```

Example:
```bash
aws ec2 create-vpc-endpoint --vpc-endpoint-type ServiceNetwork --vpc-id vpc-007165e10c4729cc0  --subnet-ids subnet-00d8da6295058fced subnet-00d60dc893a32243d  --security-group-ids sg-0f6fac5a45e5f093a  --service-network-arn arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7
```

Output:
```json
{
    "VpcEndpoint": {
        "VpcEndpointId": "vpce-0da9c8491fcf3a6d9",
        "VpcEndpointType": "ServiceNetwork",
        "VpcId": "vpc-007165e10c4729cc0",
        "State": "Pending",
        "SubnetIds": [
            "subnet-00d8da6295058fced",
            "subnet-00d60dc893a32243d"
        ],
        "Groups": [
            {
                "GroupId": "sg-0f6fac5a45e5f093a",
                "GroupName": "providervpc-LocalTrafficSecurityGroup-kphu4qAaarWp"
            }
        ],
        "IpAddressType": "IPV4",
        "PrivateDnsEnabled": false,
        "CreationTimestamp": "2025-03-22T07:26:50.955000+00:00",
        "Tags": [],
        "OwnerId": "214517638895",
        "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7"
    }
}
```

## Testing Connectivity

Now that the service network is deployed and the Resource Configurations are associated, it's time to test the solution end to end.

1. Find out the FQDN for the customer side resources through the service network endpoint details in the SaaS provider account. Start with describing the VPC endpoints using the following command in the SaaS provider account. Make sure your terminal still has the credentials for the SaaS provider account active.

```bash
aws ec2 describe-vpc-endpoint-associations
```

The example output below shows the DNS names assigned to each resource. Make note of these DNS names and their corresponding customer account IDs, which can be found in the AssociatedResourceArn field of each resource. This mapping helps identify which DNS name belongs to which customer's resource.

```json
{
    "VpcEndpointAssociations": [
        {
            "Id": "vpce-rsc-asc-06af3511162bf4919",
            "VpcEndpointId": "vpce-0da9c8491fcf3a6d9",
            "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7",
            "ServiceNetworkName": "providersne",
            "AssociatedResourceAccessibility": "Accessible",
            "DnsEntry": {
                "DnsName": "vpce-0da9c8491fcf3a6d9-snra-0eae7f31d03c50d72.rcfg-01ae20d51100803bd.4232ccc.vpc-lattice-rsc.us-east-1.on.aws",
                "HostedZoneId": "Z08285483B41F6ASEM5U1"
            },
            "AssociatedResourceArn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd"
        },
        {
            "Id": "vpce-rsc-asc-0cb29006631b3cdca",
            "VpcEndpointId": "vpce-0da9c8491fcf3a6d9",
            "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7",
            "ServiceNetworkName": "providersne",
            "AssociatedResourceAccessibility": "Accessible",
            "DnsEntry": {
                "DnsName": "vpce-0da9c8491fcf3a6d9-snra-012648bc4e8ad80dc.rcfg-0349a6320d67a56ad.4232ccc.vpc-lattice-rsc.us-east-1.on.aws",
                "HostedZoneId": "Z08285483B41F6ASEM5U1"
            },
            "AssociatedResourceArn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad"
        }
    ]
}
```

2. Log on to the EC2 instance in the SaaS provider VPC (for example with Session Manager) and initiate traffic using the DNS Name from the previous step. You should get a successful connection as in the example below. If you remember, we are using a sample application listening on TCP 1234 to simulate a resource in customer VPC.

```bash
sh-4.2$ telnet vpce-0da9c8491fcf3a6d9-snra-0eae7f31d03c50d72.rcfg-01ae20d51100803bd.4232ccc.vpc-lattice-rsc.us-east-1.on.aws 1234
Trying 10.0.1.178...
Connected to vpce-0da9c8491fcf3a6d9-snra-0eae7f31d03c50d72.rcfg-01ae20d51100803bd.4232ccc.vpc-lattice-rsc.us-east-1.on.aws.
Escape character is '^]'.
Connection successful!
Connection closed by foreign host.
sh-4.2$
```

Next, connect to the second customer instance in the same manner. This concludes the walkthrough. You can enhance this solution further by automating the steps shown in this blog and using a Private Hosted Zone to generate more user friendly hostnames, instead of the system-generated DNS names.

## Conclusion

Resource Gateways and VPC Lattice provide a powerful solution for SaaS providers to securely access customer resources in multi-tenant architectures. By leveraging these technologies, you can overcome traditional networking challenges, maintain strict security boundaries, and scale your connectivity to thousands of customers.

The architecture you've just built offers a flexible foundation that works well for most multi-tenant services. It enables you to securely access customer resources across account boundaries while simplifying network management and reducing operational complexity. The solution's ability to scale horizontally accommodates growing numbers of tenants, and its design maintains clear traffic separation between inbound and outbound flows.
TBD- . Below are the steps to deploy the solution in your own AWS environment.

## Prerequisites

Following resources are required to walk through the steps:

* Preferably 3 separate AWS accounts to simulate one SaaS provider and two customers. For testing purposes you can also use the same account for SaaS provider and customers and skip the resource sharing parts in the instructions. We use 3 separate AWS accounts in the steps to keep it as close as possible to real world scenarios. 
* One VPC per account, with an EC2 instance in the SaaS provider VPC to simulate the Service, and a resource in each customer VPC (such as a database or an EC2 instance listening on a TCP port) to simulate the customer resource. In this blog post we used a sample TCP app listening on TCP port 1234. We will be using this CloudFormation template to deploy the base infrastructure.
* AWS Command Line Interface (AWS CLI): You need AWS CLI installed and configured on the workstation from where you are going to try the steps mentioned below.
* Access to credentials to be configured in AWS CLI, which should have the required IAM permissions to spin up and modify the resources mentioned in this post for each of the 3 accounts mentioned above.
* This blog post assumes that us-east-1 Region is used and your AWS CLI default Region is us-east-1. If us-east-1 is not the default Region, please mentioned the Regions explicitly while executing AWS CLI commands using --region us-east-1 switch.

### Considerations:

* The compute platform doesn't matter as long as it sits in the VPC. We will be using an EC2 instance to showcase connectivity between service and resource. 
* For zonal consistency between SaaS provider VPC and customer VPCs make sure that you are using all AZs in the SaaS provider side VPC for service network endpoints. 
* In the SaaS provider side VPC, we recommend a dedicated subnet for the service network endpoints in every AZ, each with one IP per resource you will expect. For example if your tenants all share one resource each and you expect no more than 100 tenants to share resources with you, /25 subnets would be sufficient to host the 100 IP addresses.

## Implementation Guide for A - Dedicated Service Network per Customer

### High level steps

#### Customer side:

1. In the customer account, create a Resource Gateway in the VPC where the resource in question is located. While creating the Resource Gateway, choose at-least two subnets across two AZs and the Security Group with required rules in it.
2. Create a Resource Configuration using the Resource Gateway created in previous step. The Resource Configuration should be created using the TCP port the resource is listening on and in resource-configuration-definition, type should be chosen as ipResource and should point to the IP address of the TCP server.
3. Create a Service Network and associate the Resource Configuration created in previous step to this Service Network. 
3. Create a Resource Share for this Service Network and share it with the SaaS provider account.
4. Repeat these steps for other customer accounts.

#### SaaS provider side:

1. In the SaaS provider account, accept the Resource Share Invitation from  customer accounts.
2. Create a service network endpoint for the service network of each customer in the SaaS provider VPC. Make sure to choose all the AZs so that AZ mismatch between customer and SaaS provider can be avoided. Alternatively you can align AZs between Resource Gateways in customer VPCs and service network endpoint in the SaaS provider VPC.
4. Obtain the DNS Names assigned by the system for each customer resource and test connectivity.
5. Optionally, create and associate a Private Hosted Zone with the SaaS provider VPC and create CNAME records to point the system-generated DNS names to more user friendly domain names.

### Detailed steps

#### Setting up the customer side

Let's begin with setting up the customer side by creating a Resource Gateway, then creating a Resource Configuration using this newly created Resource Gateway and resource/s to be exposed to the SaaS provider. First download this CloudFormation template ([RGBlog.yaml](https://gitlab.aws.dev/vijaym/LatticeResourceGWBlog/-/raw/main/RGBlog.yaml?ref_type=heads&inline=false)) which we will use to deploy the base infrastructure in all accounts.


##### Customer 1:

1. Switch Roles or use credentials to log on to the 1st customer account from your terminal.
2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name consumer1vpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Service-Consumer1 \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id such as one shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:001182731686:stack/consumer1vpc/28effa40-06d2-11f0-adb6-122d7d2f3903"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name consumer1vpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk:

```bash
aws cloudformation list-stack-resources --stack-name consumer1vpc > consumer1vpc.json
```

The file created in this step will have content similar to the one shown in the snippet below (we have removed the resources that are not relevant to the instructions):

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-0535f65e2e5db2c36",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-06-13T08:21:42.255000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-0e729e42439b39c00",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-06-13T08:22:12.909000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-0c7251d3cd3580f4c",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-06-13T08:21:32.822000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-07ed64c78bf050e3e",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:21:58.023000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-0bf45cfa188d0190d",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:21:58.471000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
```

4. Create a Resource Gateway using the above information:

```bash
aws vpc-lattice create-resource-gateway --name consumer1gateway --vpc-identifier vpc-0c7251d3cd3580f4c --subnet-ids subnet-0bf45cfa188d0190d subnet-07ed64c78bf050e3e --security-group-ids sg-0535f65e2e5db2c36 --ip-address-type IPV4
```

The output will look like this (make a note of the Resource Gateway arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourcegateway/rgw-06745d5fa17e71678",
    "id": "rgw-06745d5fa17e71678",
    "ipAddressType": "IPV4",
    "name": "consumer1gateway",
    "securityGroupIds": [
        "sg-0535f65e2e5db2c36"
    ],
    "status": "ACTIVE",
    "subnetIds": [
        "subnet-0bf45cfa188d0190d",
        "subnet-07ed64c78bf050e3e"
    ],
    "vpcIdentifier": "vpc-0c7251d3cd3580f4c"
}
```

5. Create a Resource Configuration using the Resource Gateway:

First, retrieve the private IP address of the TCP server:

```bash
aws ec2 describe-instances --instance-ids i-0e729e42439b39c00 --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
```

Output (make a note of it):
```
10.0.1.14
```

Then create the Resource Configuration:

```bash
aws vpc-lattice create-resource-configuration --name consumer1resourceconfig --resource-gateway-identifier rgw-06745d5fa17e71678  --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=10.0.1.14}'
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "allowAssociationToShareableServiceNetwork": true,
    "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-03eb9d30003e44cbc",
    "id": "rcfg-03eb9d30003e44cbc",
    "name": "consumer1resourceconfig",
    "portRanges": [
        "1234"
    ],
    "protocol": "TCP",
    "resourceConfigurationDefinition": {
        "ipResource": {
            "ipAddress": "10.0.1.14"
        }
    },
    "resourceGatewayId": "rgw-06745d5fa17e71678",
    "status": "ACTIVE",
    "type": "SINGLE"
}
```

6. Create a Service Network with '--sharing-config' flag as 'enabled=true'

```bash
aws vpc-lattice create-service-network --name cx1-service-network --sharing-config enabled=true
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetworkresourceassociation/snra-08ab290695bf1e2f5",
    "createdBy": "661497052113",
    "id": "snra-08ab290695bf1e2f5",
    "status": "CREATE_IN_PROGRESS"
}
```
7. Associate the Resource Configuration from step 5 with Service Network from step 6.

```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier rcfg-02068184d728a2883 --service-network-identifier sn-0a414e0c612df5a26
```

Output will be similar to this

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetworkresourceassociation/snra-06dd45332f4d40737",
    "createdBy": "001182731686",
    "id": "snra-06dd45332f4d40737",
    "status": "CREATE_IN_PROGRESS"
}
```


8. Create a Resource Share for SaaS provider account by providing SaaS provider account id as the value of '--principal' switch.

```bash
aws ram create-resource-share --name consumer1sn --principals 214517638895
```

The output will be similar to this (make a note of the arn):

```json
{
    "resourceShare": {
        "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21",
        "name": "consumer1sn",
        "owningAccountId": "661497052113",
        "allowExternalPrincipals": true,
        "status": "ACTIVE",
        "tags": [],
        "creationTime": "2025-06-14T09:24:35.985000+08:00",
        "lastUpdatedTime": "2025-06-14T09:24:35.985000+08:00"
    }
}
```

9. Associate the Service Network arn from step 6 with the resource share arn from step 8.

```bash
aws ram associate-resource-share --resource-share arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21 --resource-arns arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b
```

The output will be similar to this (make a note of the arn):

```json
{
    "resourceShareAssociations": [
        {
            "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21",
            "associatedEntity": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b",
            "associationType": "RESOURCE",
            "status": "ASSOCIATING",
            "external": false
        }
    ]
}
```

##### Customer 2 Setup

1. Switch Roles or use credentials to log on to the second customer account from your terminal.

2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name consumer2vpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Service-Consumer2 \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id such as one shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:001182731686:stack/consumer2vpc/28effa40-06d2-11f0-adb6-122d7d2f3903"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name consumer2vpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk:

```bash
aws cloudformation list-stack-resources --stack-name consumer2vpc > consumer2vpc.json
```

The file created in this step will have content similar to the one shown in the snippet below (we have removed the resources that are not relevant to the instructions):

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-09e324e0e2fbab9f6",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-06-13T08:23:02.564000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-0b8a4392dc722b7a7",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-06-13T08:23:32.126000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-074bcdd680fdae720",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-06-13T08:22:52.625000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-08b91245a53c4d631",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:23:17.215000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-01a5eba50b4ba6dde",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:23:17.215000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
```

4. Create a Resource Gateway using the above information:

```bash
aws vpc-lattice create-resource-gateway --name consumer2gateway --vpc-identifier vpc-074bcdd680fdae720 --subnet-ids subnet-08b91245a53c4d631 subnet-01a5eba50b4ba6dde --security-group-ids sg-09e324e0e2fbab9f6 --ip-address-type IPV4
```

The output will look like this (make a note of the Resource Gateway arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourcegateway/rgw-01da2681a040c1d94",
    "id": "rgw-01da2681a040c1d94",
    "ipAddressType": "IPV4",
    "name": "consumer2gateway",
    "securityGroupIds": [
        "sg-09e324e0e2fbab9f6"
    ],
    "status": "ACTIVE",
    "subnetIds": [
        "subnet-08b91245a53c4d631",
        "subnet-01a5eba50b4ba6dde"
    ],
    "vpcIdentifier": "vpc-074bcdd680fdae720"
}
```

5. Create a Resource Configuration using the Resource Gateway:

First, retrieve the private IP address of the TCP server:

```bash
aws ec2 describe-instances --instance-ids i-0b8a4392dc722b7a7 --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
```

Output (make a note of it):
```
10.0.1.157
```

Then create the Resource Configuration:

```bash
aws vpc-lattice create-resource-configuration --name consumer2resourceconfig --resource-gateway-identifier rgw-01da2681a040c1d94 --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=10.0.1.157}'
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "allowAssociationToShareableServiceNetwork": true,
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-02068184d728a2883",
    "id": "rcfg-02068184d728a2883",
    "name": "consumer2resourceconfig",
    "portRanges": [
        "1234"
    ],
    "protocol": "TCP",
    "resourceConfigurationDefinition": {
        "ipResource": {
            "ipAddress": "10.0.1.157"
        }
    },
    "resourceGatewayId": "rgw-01da2681a040c1d94",
    "status": "ACTIVE",
    "type": "SINGLE"
}
```

6. Create a Service Network with '--sharing-config' flag as 'enabled=true'

```bash
aws vpc-lattice create-service-network --name cx2-service-network --sharing-config enabled=true
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26",
    "authType": "NONE",
    "id": "sn-0a414e0c612df5a26",
    "name": "cx2-service-network",
    "sharingConfig": {
        "enabled": true
    }
}
```

7. Associate the Resource Configuration from step 5 with Service Network from step 6.

```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier rcfg-02068184d728a2883 --service-network-identifier sn-0a414e0c612df5a26
```

Output will be similar to this

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetworkresourceassociation/snra-06dd45332f4d40737",
    "createdBy": "001182731686",
    "id": "snra-06dd45332f4d40737",
    "status": "CREATE_IN_PROGRESS"
}
```


8. Create a Resource Share for SaaS provider account by providing SaaS provider account id as the value of '--principal' switch.

```bash
aws ram create-resource-share --name consumer2sn --principals 214517638895
```

The output will be similar to this (make a note of the arn):

```json
{
    "resourceShare": {
        "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42",
        "name": "consumer2sn",
        "owningAccountId": "001182731686",
        "allowExternalPrincipals": true,
        "status": "ACTIVE",
        "tags": [],
        "creationTime": "2025-06-14T09:29:46.558000+08:00",
        "lastUpdatedTime": "2025-06-14T09:29:46.558000+08:00"
    }
}
```

9. Associate the Service Network arn from step 6 with the resource share arn from step 8.

```bash
aws ram associate-resource-share --resource-share arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42 --resource-arns arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26
```

The output will be similar to this (make a note of the arn):

```json
{
    "resourceShareAssociations": [
        {
            "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42",
            "associatedEntity": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26",
            "associationType": "RESOURCE",
            "status": "ASSOCIATING",
            "external": false
        }
    ]
}
```

# Setting up the SaaS Provider Side

1. Switch roles or use credentials to log on to the SaaS provider account from your terminal.

2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name providervpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Provider \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id such as the one shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:214517638895:stack/providervpc/6e2e4950-06e5-11f0-8245-1207e77f631b"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name <name of your stack from above step> | grep StackStatus
```

Example:
```bash
aws cloudformation describe-stacks --stack-name providervpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk. We will be using these IDs throughout this blog post.

```bash
aws cloudformation list-stack-resources --stack-name providervpc > providervpc.json 
```

The file created in this step will have content similar to the one shown in the snippet below. We have removed the resources not being used in the instructions:

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-0fa7fce98345c9604",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-06-13T08:24:19.336000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-0693552b44c4e995f",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-06-13T08:24:50.228000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-069a64174df6950b7",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-06-13T08:24:09.488000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-0e164f16b26e7a329",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:24:34.964000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-0730b2578a2456c07",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-06-13T08:24:37.084000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
```

4. Obtain the Resource Share Invitation arns shared by the two customer accounts. We need access to the Resource Shares to retrieve the Resource Configurations that were shared.

```bash
aws ram get-resource-share-invitations
```

Output example below, make note of the resourceShareInvitationArns:

```json
{
    "resourceShareInvitations": [
        {
            "resourceShareInvitationArn": "arn:aws:ram:us-east-1:661497052113:resource-share-invitation/0124cfbf-ebda-4bbf-bcb8-b3ac487630c1",
            "resourceShareName": "consumer1sn",
            "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21",
            "senderAccountId": "661497052113",
            "receiverAccountId": "214517638895",
            "invitationTimestamp": "2025-06-14T09:24:36.979000+08:00",
            "status": "PENDING"
        },
        {
            "resourceShareInvitationArn": "arn:aws:ram:us-east-1:001182731686:resource-share-invitation/5ff2ac29-0bff-44f0-ac91-92a2cf656218",
            "resourceShareName": "consumer2sn",
            "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42",
            "senderAccountId": "001182731686",
            "receiverAccountId": "214517638895",
            "invitationTimestamp": "2025-06-14T09:29:46.760000+08:00",
            "status": "PENDING"
        }
    ]
}
```

5. Accept both Resource Share invitations using the following command one at a time, so that we can use those Resource Configuration with our service network in subsequent steps.

```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn <resourceShareInvitationArn from step 3>
```

Example:
```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn arn:aws:ram:us-east-1:661497052113:resource-share-invitation/0124cfbf-ebda-4bbf-bcb8-b3ac487630c1
```

Output:
```json
{
    "resourceShareInvitation": {
        "resourceShareInvitationArn": "arn:aws:ram:us-east-1:661497052113:resource-share-invitation/0124cfbf-ebda-4bbf-bcb8-b3ac487630c1",
        "resourceShareName": "consumer1sn",
        "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/aee02db8-1ea3-4404-a8bd-c915e113ad21",
        "senderAccountId": "661497052113",
        "receiverAccountId": "214517638895",
        "invitationTimestamp": "2025-06-14T09:33:56.662000+08:00",
        "status": "ACCEPTED"
    }
}
```

```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn arn:aws:ram:us-east-1:001182731686:resource-share-invitation/5ff2ac29-0bff-44f0-ac91-92a2cf656218
```

Output:
```json
{
    "resourceShareInvitation": {
        "resourceShareInvitationArn": "arn:aws:ram:us-east-1:001182731686:resource-share-invitation/5ff2ac29-0bff-44f0-ac91-92a2cf656218",
        "resourceShareName": "consumer2sn",
        "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/a782a18e-9753-4984-9ca4-73adcda3ac42",
        "senderAccountId": "001182731686",
        "receiverAccountId": "214517638895",
        "invitationTimestamp": "2025-06-14T09:34:14.828000+08:00",
        "status": "ACCEPTED"
    }
}
```

6. List the shared Service Networks for customer 1 and 2 and make not of the respective ARNs.

```bash
aws vpc-lattice list-service-networks
```

Output example below, make note of the ARNs:

```json
{
    "items": [
        {
            "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26",
            "createdAt": "2025-06-14T01:27:51.920000+00:00",
            "id": "sn-0a414e0c612df5a26",
            "lastUpdatedAt": "2025-06-14T01:27:51.920000+00:00",
            "name": "cx2-service-network",
            "numberOfAssociatedResourceConfigurations": 1,
            "numberOfAssociatedServices": 0,
            "numberOfAssociatedVPCs": 0
        },
        {
            "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b",
            "createdAt": "2025-06-14T01:23:08.249000+00:00",
            "id": "sn-0f7a8ceee967ecf4b",
            "lastUpdatedAt": "2025-06-14T01:23:08.249000+00:00",
            "name": "cx1-service-network",
            "numberOfAssociatedResourceConfigurations": 1,
            "numberOfAssociatedServices": 0,
            "numberOfAssociatedVPCs": 0
        }
    ]
}
```

7. Create Service Network Endpoints for customer 1 in provider VPC using the Service Network ARNs from the previous steps and other parameters form step 3.

```bash
aws ec2 create-vpc-endpoint --vpc-endpoint-type ServiceNetwork --vpc-id vpc-069a64174df6950b7 --subnet-ids subnet-0e164f16b26e7a329 subnet-0730b2578a2456c07  --service-network-arn arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b --security-group-ids sg-0fa7fce98345c9604
```

Output will be similar to the one below. Make note of the "VpcEndpointId"

```json
{
    "VpcEndpoint": {
        "VpcEndpointId": "vpce-0057aabf75896925f",
        "VpcEndpointType": "ServiceNetwork",
        "VpcId": "vpc-069a64174df6950b7",
        "State": "Pending",
        "SubnetIds": [
            "subnet-0e164f16b26e7a329",
            "subnet-0730b2578a2456c07"
        ],
        "Groups": [
            {
                "GroupId": "sg-0fa7fce98345c9604",
                "GroupName": "onesnpercx-LocalTrafficSecurityGroup-GLcJ2TGuYPt3"
            }
        ],
        "IpAddressType": "IPV4",
        "PrivateDnsEnabled": false,
        "CreationTimestamp": "2025-06-14T01:52:18.467000+00:00",
        "Tags": [],
        "OwnerId": "214517638895",
        "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b"
    }
}
```

8. Create Service Network Endpoints for customer 2 in provider VPC using the Service Network ARNs from the previous steps and other parameters form step 3.

```bash
aws ec2 create-vpc-endpoint --vpc-endpoint-type ServiceNetwork --vpc-id vpc-069a64174df6950b7 --subnet-ids subnet-0e164f16b26e7a329 subnet-0730b2578a2456c07  --service-network-arn arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26 --security-group-ids sg-0fa7fce98345c9604
```

Output will be similar to the one below. Make note of the "VpcEndpointId"

```json
{
    "VpcEndpoint": {
        "VpcEndpointId": "vpce-0f067bfe89cb48bb2",
        "VpcEndpointType": "ServiceNetwork",
        "VpcId": "vpc-069a64174df6950b7",
        "State": "Pending",
        "SubnetIds": [
            "subnet-0e164f16b26e7a329",
            "subnet-0730b2578a2456c07"
        ],
        "Groups": [
            {
                "GroupId": "sg-0fa7fce98345c9604",
                "GroupName": "onesnpercx-LocalTrafficSecurityGroup-GLcJ2TGuYPt3"
            }
        ],
        "IpAddressType": "IPV4",
        "PrivateDnsEnabled": false,
        "CreationTimestamp": "2025-06-14T01:53:26.456000+00:00",
        "Tags": [],
        "OwnerId": "214517638895",
        "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26"
    }
}
```

## Testing Connectivity

Now that the service network endpoints for each customer is created in SaaS provider VPC, it's time to test the solution end to end.

1. Find out the FQDN for the customer side resources through the service network endpoint details in the SaaS provider account. Start with describing the VPC endpoints using the following command in the SaaS provider account. Make sure your terminal still has the credentials for the SaaS provider account active.

```bash
aws ec2 describe-vpc-endpoint-associations
```

The example output below shows the DNS names assigned to each resource. Make note of these DNS names and their corresponding customer service networks, which can be found in "DnsName" under "DnsEntry" coresponding to each "ServiceNetworkName". This mapping helps identify which DNS name belongs to which customer's resource.

``

Output will be similar to the one below. Make note of the value of "DnsName" under "DnsEntry" coresponding to each "ServiceNetworkName".

```json
aws ec2 describe-vpc-endpoint-associations --vpc-endpoint-ids vpce-0057aabf75896925f vpce-0f067bfe89cb48bb2
{
    "VpcEndpointAssociations": [
        {
            "Id": "vpce-rsc-asc-03f2f67b4d6a5e7d5",
            "VpcEndpointId": "vpce-0057aabf75896925f",
            "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:661497052113:servicenetwork/sn-0f7a8ceee967ecf4b",
            "ServiceNetworkName": "cx1-service-network",
            "AssociatedResourceAccessibility": "Accessible",
            "DnsEntry": {
                "DnsName": "vpce-0057aabf75896925f-snra-08ab290695bf1e2f5.rcfg-03eb9d30003e44cbc.4232ccc.vpc-lattice-rsc.us-east-1.on.aws",
                "HostedZoneId": "Z08285483B41F6ASEM5U1"
            },
            "AssociatedResourceArn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-03eb9d30003e44cbc"
        },
        {
            "Id": "vpce-rsc-asc-078d3d60c3fbba6b8",
            "VpcEndpointId": "vpce-0f067bfe89cb48bb2",
            "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:001182731686:servicenetwork/sn-0a414e0c612df5a26",
            "ServiceNetworkName": "cx2-service-network",
            "AssociatedResourceAccessibility": "Accessible",
            "DnsEntry": {
                "DnsName": "vpce-0f067bfe89cb48bb2-snra-06dd45332f4d40737.rcfg-02068184d728a2883.4232ccc.vpc-lattice-rsc.us-east-1.on.aws",
                "HostedZoneId": "Z08285483B41F6ASEM5U1"
            },
            "AssociatedResourceArn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-02068184d728a2883"
        }
    ]
}
```

2. Log on to the EC2 instance in the SaaS provider VPC (for example with Session Manager) and initiate traffic using the DNS Name from the previous step. You should get a successful connection as in the example below. If you remember, we are using a sample application listening on TCP 1234 to simulate a resource in customer VPC.

```bash
sh-4.2$ telnet vpce-0057aabf75896925f-snra-08ab290695bf1e2f5.rcfg-03eb9d30003e44cbc.4232ccc.vpc-lattice-rsc.us-east-1.on.aws 1234
Trying 10.0.1.210...
Connected to vpce-0057aabf75896925f-snra-08ab290695bf1e2f5.rcfg-03eb9d30003e44cbc.4232ccc.vpc-lattice-rsc.us-east-1.on.aws.
Escape character is '^]'.
Connection successful!
Connection closed by foreign host.
sh-4.2$
sh-4.2$
sh-4.2$
sh-4.2$ telnet vpce-0f067bfe89cb48bb2-snra-06dd45332f4d40737.rcfg-02068184d728a2883.4232ccc.vpc-lattice-rsc.us-east-1.on.aws 1234
Trying 10.0.1.40...
Connected to vpce-0f067bfe89cb48bb2-snra-06dd45332f4d40737.rcfg-02068184d728a2883.4232ccc.vpc-lattice-rsc.us-east-1.on.aws.
Escape character is '^]'.
Connection successful!
Connection closed by foreign host.
sh-4.2$
```

This concludes the implementation steps for one Service Network per customer option. Next section covers the implementation steps for shared Service Network option.

--------------


## Implementation Guide for B - shared Service Network

### High level steps

#### Customer side:

1. In the customer account, create a Resource Gateway in the VPC where the resource in question is located. While creating the Resource Gateway, choose at-least two subnets across two AZs and the Security Group with required rules in it.
2. Create a Resource Configuration using the Resource Gateway created in previous step. The Resource Configuration should be created using the TCP port the resource is listening on and in resource-configuration-definition, type should be chosen as ipResource and should point to the IP address of the TCP server.
3. Create a Resource Share for this Resource Configuration and share it with the SaaS provider account.
4. Repeat these steps for other customer accounts.

#### SaaS provider side:

1. In the SaaS provider account, accept the Resource Share Invitation from all the customer accounts.
2. Create a service network with --sharing-config enabled=false . By default, sharing config is set to true, which will not allow shared Resource Configuration association with the service network.
3. Create a service network endpoint for the service network in the SaaS provider VPC. Make sure to choose all the AZs so that AZ mismatch between customer and SaaS provider can be avoided. Alternatively you can align AZs between Resource Gateways in customer VPCs and service network endpoint in the SaaS provider VPC.
4. Obtain the DNS Names assigned by the system for each customer resource and test connectivity.
5. Optionally, create and associate a Private Hosted Zone with the SaaS provider VPC and create CNAME records to point the system-generated DNS names to more user.friendly domain names.

### Detailed steps

#### Setting up the customer side

Let's begin with setting up the customer side by creating a Resource Gateway, then creating a Resource Configuration using this newly created Resource Gateway and resource/s to be exposed to the SaaS provider. First download this CloudFormation template ([RGBlog.yaml](https://gitlab.aws.dev/vijaym/LatticeResourceGWBlog/-/raw/main/RGBlog.yaml?ref_type=heads&inline=false)) which we will use to deploy the base infrastructure in all accounts.



##### Customer 1:

1. Switch Roles or use credentials to log on to the 1st customer account from your terminal.
2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name consumer1vpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Service-Consumer1 \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id as shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:001182731686:stack/consumer1vpc/28effa40-06d2-11f0-adb6-122d7d2f3903"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name consumer1vpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk:

```bash
aws cloudformation list-stack-resources --stack-name consumer1vpc > consumer1vpc.json
```

The file created in this step will have content similar to the one shown in the snippet below (we have removed the resources that are not relevant to the instructions):

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-09632e5c873e108cd",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-03-22T04:00:40.052000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-08a56ca3848b640b7",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-03-22T04:01:55.275000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-0b0e03534b5f3f8ef",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-03-22T04:00:30.691000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-036da24d3897feb13",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T04:00:35.826000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-00921cc39600a34ec",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T04:00:35.756000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
    ]
}
```

4. Create a Resource Gateway using the above information:

```bash
aws vpc-lattice create-resource-gateway --name consumer1gateway --vpc-identifier vpc-0b0e03534b5f3f8ef --subnet-ids subnet-036da24d3897feb13 subnet-00921cc39600a34ec --security-group-ids sg-09632e5c873e108cd --ip-address-type IPV4
```

The output will look like this (make a note of the Resource Gateway arn and id):

```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourcegateway/rgw-036d78f254aaa2dcf",
    "id": "rgw-036d78f254aaa2dcf",
    "ipAddressType": "IPV4",
    "name": "consumer1gateway",
    "securityGroupIds": [
        "sg-09632e5c873e108cd"
    ],
    "status": "ACTIVE",
    "subnetIds": [
        "subnet-036da24d3897feb13",
        "subnet-00921cc39600a34ec"
    ],
    "vpcIdentifier": "vpc-0b0e03534b5f3f8ef"
}
```

5. Create a Resource Configuration using the Resource Gateway:

First, retrieve the private IP address of the TCP server:

```bash
aws ec2 describe-instances --instance-ids i-08a56ca3848b640b7 --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
```

Output (make a note of it):
```
10.0.1.121
```

Then create the Resource Configuration:

```bash
aws vpc-lattice create-resource-configuration --no-allow-association-to-shareable-service-network --name consumer1resourceconfig --resource-gateway-identifier rgw-036d78f254aaa2dcf  --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=10.0.1.121}'
```

The output will be similar to this (make a note of the arn and id):

```json
{
    "allowAssociationToShareableServiceNetwork": false,
    "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad",
    "id": "rcfg-0349a6320d67a56ad",
    "name": "consumer1resourceconfig",
    "portRanges": [
        "1234"
    ],
    "protocol": "TCP",
    "resourceConfigurationDefinition": {
        "ipResource": {
            "ipAddress": "10.0.1.121"
        }
    },
    "resourceGatewayId": "rgw-036d78f254aaa2dcf",
    "status": "ACTIVE",
    "type": "SINGLE"
}
```

6. Create a Resource Share using AWS RAM and share it with the SaaS provider account:

```bash
aws ram create-resource-share --name consumer1rcfg --principals 214517638895
```

Output:
```json
{
    "resourceShare": {
        "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d",
        "name": "consumer1rcfg",
        "owningAccountId": "001182731686",
        "allowExternalPrincipals": true,
        "status": "ACTIVE",
        "tags": [],
        "creationTime": "2025-03-22T05:36:57.300000+00:00",
        "lastUpdatedTime": "2025-03-22T05:36:57.300000+00:00"
    }
}
```

7. Associate the Resource Configuration with the Resource Share:

```bash
aws ram associate-resource-share --resource-share arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d --resource-arns arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad
```

Output:
```json
{
    "resourceShareAssociations": [
        {
            "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d",
            "associatedEntity": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad",
            "associationType": "RESOURCE",
            "status": "ASSOCIATING",
            "external": false
        }
    ]
}
```

##### Customer 2 Setup

1. Switch Roles or use credentials to log on to the second customer account from your terminal.

2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name consumer2vpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Service-Consumer2 \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id as shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:661497052113:stack/consumer2vpc/da5e6c60-06d4-11f0-b7b3-0e7cb26ba367"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name <name of your stack from above step> | grep StackStatus
```

Example:
```bash
aws cloudformation describe-stacks --stack-name consumer2vpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk. We will be using these IDs throughout this blog post.

```bash
aws cloudformation list-stack-resources --stack-name consumer2vpc > consumer2vpc.json 
```

The file created in this step will have content similar to the one shown in the snippet below. We have removed the resources not being used in the instructions:

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-043275505a56c7210",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-03-22T04:19:55.396000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "TCPServerInstance",
            "PhysicalResourceId": "i-05b8b6e9ef1d17db8",
            "ResourceType": "AWS::EC2::Instance",
            "LastUpdatedTimestamp": "2025-03-22T04:21:10.350000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-0fd531b96013cb156",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-03-22T04:19:46.622000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-0bc5b9aa3bec7eee4",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T04:19:51.546000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-0f04b9643e598a98a",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T04:19:53.172000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
    ]
}
```

4. Create a Resource Gateway using the above information using the below command:

```bash
aws vpc-lattice create-resource-gateway --name consumer2gateway --vpc-identifier <vpc-id from step2.1> --subnet-ids <private-subnet-1 id from step 2.a> <private-subnet-2 id from step 2.a> --security-group-ids <LocalTrafficSecurityGroup id from step 2.a> --ip-address-type IPV4
```

For Example:
```bash
aws vpc-lattice create-resource-gateway --name consumer2gateway --vpc-identifier vpc-0fd531b96013cb156 --subnet-ids subnet-0bc5b9aa3bec7eee4 subnet-0f04b9643e598a98a --security-group-ids sg-043275505a56c7210 --ip-address-type IPV4
```

Output will look like the below example, make a note of Resource Gateway arn and id from the output to be used in next step:
```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourcegateway/rgw-0c2ad17dc18ff5bbf",
    "id": "rgw-0c2ad17dc18ff5bbf",
    "ipAddressType": "IPV4",
    "name": "consumer1gateway",
    "securityGroupIds": [
        "sg-043275505a56c7210"
    ],
    "status": "ACTIVE",
    "subnetIds": [
        "subnet-0bc5b9aa3bec7eee4",
        "subnet-0f04b9643e598a98a"
    ],
    "vpcIdentifier": "vpc-0fd531b96013cb156"
}
```

5. Create Resource Configuration using the Resource Gateway created in step 3 and IP address of the TCP server to be used as resource. For that first we need to find the IP address of the EC2 instance running the TCP server.

    a. Find the IP address of the TCP server using the below command:

    ```bash
    aws ec2 describe-instances --instance-ids <instance-id from step 2.a> --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
    ```

    Example:
    ```bash
    aws ec2 describe-instances --instance-ids i-05b8b6e9ef1d17db8 --query 'Reservations[*].Instances[*].PrivateIpAddress' --output text
    ```

    Output (make a note of it):
    ```
    10.0.1.43
    ```

    b. Create Resource Configuration using the command below:

    ```bash
    aws vpc-lattice create-resource-configuration --no-allow-association-to-shareable-service-network --name consumer2resourceconfig --resource-gateway-identifier <resource gateway id from step 3>  --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=<ip address from step 4. above>}'
    ```

    Example:
    ```bash
    aws vpc-lattice create-resource-configuration --no-allow-association-to-shareable-service-network --name consumer2resourceconfig --resource-gateway-identifier rgw-0c2ad17dc18ff5bbf  --type SINGLE --protocol TCP --port-ranges 1234 --resource-configuration-definition 'ipResource={ipAddress=10.0.1.43}'
    ```

    Output will be similar to one shown below, make a note of arn and id for the Resource Configuration from the output:
    ```json
    {
        "allowAssociationToShareableServiceNetwork": false,
        "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd",
        "id": "rcfg-01ae20d51100803bd",
        "name": "consumer2resourceconfig",
        "portRanges": [
            "1234"
        ],
        "protocol": "TCP",
        "resourceConfigurationDefinition": {
            "ipResource": {
                "ipAddress": "10.0.1.43"
            }
        },
        "resourceGatewayId": "rgw-0c2ad17dc18ff5bbf",
        "status": "ACTIVE",
        "type": "SINGLE"
    }
    ```

6. Create a Resource share using AWS RAM and share the Resource Configuration with the SaaS provider account.

```bash
aws ram create-resource-share --name <resource-share-name> --principals <provider-account-id>
```

Example:
```bash
aws ram create-resource-share --name consumer2rcfg --principals 214517638895
```

Output:
```json
{
    "resourceShare": {
        "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185",
        "name": "consumer2rcfg",
        "owningAccountId": "661497052113",
        "allowExternalPrincipals": true,
        "status": "ACTIVE",
        "tags": [],
        "creationTime": "2025-03-22T06:04:56.208000+00:00",
        "lastUpdatedTime": "2025-03-22T06:04:56.208000+00:00"
    }
}
```

7. Associate Resource Configuration with the Resource Share created in above step.

```bash
aws ram associate-resource-share --resource-share <resource share arn from step 5 above> --resource-arns <resource configuration arn from step 4.b>
```

Example:
```bash
aws ram associate-resource-share --resource-share arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185 --resource-arns arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd 
```

Output:
```json
{
    "resourceShareAssociations": [
        {
            "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185",
            "associatedEntity": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd",
            "associationType": "RESOURCE",
            "status": "ASSOCIATING",
            "external": false
        }
    ]
}
```

Now that we have setup the customer environments successfully, let's move on to setting up the SaaS provider side of the solution.

# Setting up the SaaS Provider Side

1. Switch roles or use credentials to log on to the SaaS provider account from your terminal.

2. Deploy the CloudFormation template:

```bash
aws cloudformation create-stack \
--stack-name providervpc \
--template-body file://ResourceGatewayLatticeBase.yaml \
--parameters ParameterKey=EnvironmentName,ParameterValue=Provider \
--capabilities CAPABILITY_NAMED_IAM
```

Output of the command will be a stack id as shown below:

```json
{
    "StackId": "arn:aws:cloudformation:us-east-1:214517638895:stack/providervpc/6e2e4950-06e5-11f0-8245-1207e77f631b"
}
```

Stack creation will take approximately 7 minutes. Check the status of the stack by executing the below command every few minutes. You should see StackStatus value as CREATE_COMPLETE.

```bash
aws cloudformation describe-stacks --stack-name <name of your stack from above step> | grep StackStatus
```

Example:
```bash
aws cloudformation describe-stacks --stack-name providervpc | grep StackStatus
```

Output:
```
"StackStatus": "CREATE_COMPLETE",
```

3. Once the stack status changes to "CREATE_COMPLETE", use the below command to store the Logical IDs and corresponding Physical IDs for all resources in a file on your local disk. We will be using these IDs throughout this blog post.

```bash
aws cloudformation list-stack-resources --stack-name providervpc > providervpc.json 
```

The file created in this step will have content similar to the one shown in the snippet below. We have removed the resources not being used in the instructions:

```json
{
    "StackResourceSummaries": [
        {
            "LogicalResourceId": "LocalTrafficSecurityGroup",
            "PhysicalResourceId": "sg-0f6fac5a45e5f093a",
            "ResourceType": "AWS::EC2::SecurityGroup",
            "LastUpdatedTimestamp": "2025-03-22T06:18:36.825000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1",
            "PhysicalResourceId": "vpc-007165e10c4729cc0",
            "ResourceType": "AWS::EC2::VPC",
            "LastUpdatedTimestamp": "2025-03-22T06:18:27.566000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet1",
            "PhysicalResourceId": "subnet-00d8da6295058fced",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T06:18:32.674000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        },
        {
            "LogicalResourceId": "Vpc1PrivateSubnet2",
            "PhysicalResourceId": "subnet-00d60dc893a32243d",
            "ResourceType": "AWS::EC2::Subnet",
            "LastUpdatedTimestamp": "2025-03-22T06:18:32.632000+00:00",
            "ResourceStatus": "CREATE_COMPLETE",
            "DriftInformation": {
                "StackResourceDriftStatus": "NOT_CHECKED"
            }
        }
    ]
}
```

4. Obtain the Resource Share Invitation arns shared by the two customer accounts. We need access to the Resource Shares to retrieve the Resource Configurations that were shared.

```bash
aws ram get-resource-share-invitations
```

Output example below, make note of the resourceShareInvitationArns:

```json
{
    "resourceShareInvitations": [
        {
            "resourceShareInvitationArn": "arn:aws:ram:us-east-1:001182731686:resource-share-invitation/1bbc4cb5-b7a3-4153-a2ce-9a3b2ba962da",
            "resourceShareName": "consumer1rcfg",
            "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d",
            "senderAccountId": "001182731686",
            "receiverAccountId": "214517638895",
            "invitationTimestamp": "2025-03-22T05:36:57.631000+00:00",
            "status": "PENDING"
        },
        {
            "resourceShareInvitationArn": "arn:aws:ram:us-east-1:661497052113:resource-share-invitation/87585b25-f486-4b42-9bc3-43b251d06ead",
            "resourceShareName": "consumer2rcfg",
            "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185",
            "senderAccountId": "661497052113",
            "receiverAccountId": "214517638895",
            "invitationTimestamp": "2025-03-22T06:04:56.502000+00:00",
            "status": "PENDING"
        }
    ]
}
```

5. Accept both Resource Share invitations using the following command one at a time, so that we can use those Resource Configuration with our service network in subsequent steps.

```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn <resourceShareInvitationArn from step 3>
```

Example:
```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn arn:aws:ram:us-east-1:001182731686:resource-share-invitation/1bbc4cb5-b7a3-4153-a2ce-9a3b2ba962da
```

Output:
```json
{
    "resourceShareInvitation": {
        "resourceShareInvitationArn": "arn:aws:ram:us-east-1:001182731686:resource-share-invitation/1bbc4cb5-b7a3-4153-a2ce-9a3b2ba962da",
        "resourceShareName": "consumer1rcfg",
        "resourceShareArn": "arn:aws:ram:us-east-1:001182731686:resource-share/6524018d-9e8c-403e-b4c8-50c1e69a413d",
        "senderAccountId": "001182731686",
        "receiverAccountId": "214517638895",
        "invitationTimestamp": "2025-03-22T06:36:25.056000+00:00",
        "status": "ACCEPTED"
    }
}
```

```bash
aws ram accept-resource-share-invitation --resource-share-invitation-arn arn:aws:ram:us-east-1:661497052113:resource-share-invitation/87585b25-f486-4b42-9bc3-43b251d06ead
```

Output:
```json
{
    "resourceShareInvitation": {
        "resourceShareInvitationArn": "arn:aws:ram:us-east-1:661497052113:resource-share-invitation/87585b25-f486-4b42-9bc3-43b251d06ead",
        "resourceShareName": "consumer2rcfg",
        "resourceShareArn": "arn:aws:ram:us-east-1:661497052113:resource-share/ed669421-1c95-4eca-89db-7bfe0643e185",
        "senderAccountId": "661497052113",
        "receiverAccountId": "214517638895",
        "invitationTimestamp": "2025-03-22T06:37:39.935000+00:00",
        "status": "ACCEPTED"
    }
}
```

6. Obtain the Resource Configuration details to be used in the service network using the below command.

```bash
aws vpc-lattice list-resource-configurations
```

From the output, make note of the id of each Resource Configuration. For example:

```json
{
    "items": [
        {
            "amazonManaged": false,
            "arn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd",
            "createdAt": "2025-03-22T06:03:47.880000+00:00",
            "id": "rcfg-01ae20d51100803bd",
            "lastUpdatedAt": "2025-03-22T06:03:47.880000+00:00",
            "name": "consumer1resourceconfig",
            "resourceGatewayId": "rgw-0c2ad17dc18ff5bbf",
            "status": "ACTIVE",
            "type": "SINGLE"
        },
        {
            "amazonManaged": false,
            "arn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad",
            "createdAt": "2025-03-22T05:27:33.546000+00:00",
            "id": "rcfg-0349a6320d67a56ad",
            "lastUpdatedAt": "2025-03-22T05:27:33.546000+00:00",
            "name": "consumer2resourceconfig",
            "resourceGatewayId": "rgw-036d78f254aaa2dcf",
            "status": "ACTIVE",
            "type": "SINGLE"
        }
    ]
}
```

7. Create a service network using the following command.

```bash
aws vpc-lattice create-service-network --name providersne --sharing-config enabled=false
```

From the output of above command, make note of the service network arn and id, for example:
```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7",
    "authType": "NONE",
    "id": "sn-06ce56215f97909e7",
    "name": "providersne",
    "sharingConfig": {
        "enabled": false
    }
}
```

8. Associate each of the Resource Configurations with the newly created service network using the following command.

```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier <Resource config id from step 5> --service-network-identifier <Service Network id from step 6>
```

Example:
```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier rcfg-01ae20d51100803bd  --service-network-identifier sn-06ce56215f97909e7 
```

Output:
```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetworkresourceassociation/snra-0eae7f31d03c50d72",
    "createdBy": "214517638895",
    "id": "snra-0eae7f31d03c50d72",
    "status": "CREATE_IN_PROGRESS"
}
```

```bash
aws vpc-lattice create-service-network-resource-association --resource-configuration-identifier rcfg-0349a6320d67a56ad --service-network-identifier sn-06ce56215f97909e7 
```

Output:
```json
{
    "arn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetworkresourceassociation/snra-012648bc4e8ad80dc",
    "createdBy": "214517638895",
    "id": "snra-012648bc4e8ad80dc",
    "status": "CREATE_IN_PROGRESS"
}
```

9. Create service network endpoints in the SaaS provider VPC so that our SaaS application in that VPC can access the customer resources.

```bash
aws ec2 create-vpc-endpoint --vpc-endpoint-type ServiceNetwork --vpc-id <vpc-id from step 2.a> --subnet-ids <private subnet 1 id from step 2.a> <private subnet 2 id from step 2.a> --security-group-ids <LocalTrafficSecurityGroup id from step 2.a> --service-network-arn <service network arn from step 6>
```

Example:
```bash
aws ec2 create-vpc-endpoint --vpc-endpoint-type ServiceNetwork --vpc-id vpc-007165e10c4729cc0  --subnet-ids subnet-00d8da6295058fced subnet-00d60dc893a32243d  --security-group-ids sg-0f6fac5a45e5f093a  --service-network-arn arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7
```

Output:
```json
{
    "VpcEndpoint": {
        "VpcEndpointId": "vpce-0da9c8491fcf3a6d9",
        "VpcEndpointType": "ServiceNetwork",
        "VpcId": "vpc-007165e10c4729cc0",
        "State": "Pending",
        "SubnetIds": [
            "subnet-00d8da6295058fced",
            "subnet-00d60dc893a32243d"
        ],
        "Groups": [
            {
                "GroupId": "sg-0f6fac5a45e5f093a",
                "GroupName": "providervpc-LocalTrafficSecurityGroup-kphu4qAaarWp"
            }
        ],
        "IpAddressType": "IPV4",
        "PrivateDnsEnabled": false,
        "CreationTimestamp": "2025-03-22T07:26:50.955000+00:00",
        "Tags": [],
        "OwnerId": "214517638895",
        "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7"
    }
}
```

## Testing Connectivity

Now that the service network is deployed and the Resource Configurations are associated, it's time to test the solution end to end.

1. Find out the FQDN for the customer side resources through the service network endpoint details in the SaaS provider account. Start with describing the VPC endpoints using the following command in the SaaS provider account. Make sure your terminal still has the credentials for the SaaS provider account active.

```bash
aws ec2 describe-vpc-endpoint-associations
```

The example output below shows the DNS names assigned to each resource. Make note of these DNS names and their corresponding customer account IDs, which can be found in the AssociatedResourceArn field of each resource. This mapping helps identify which DNS name belongs to which customer's resource.

```json
{
    "VpcEndpointAssociations": [
        {
            "Id": "vpce-rsc-asc-06af3511162bf4919",
            "VpcEndpointId": "vpce-0da9c8491fcf3a6d9",
            "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7",
            "ServiceNetworkName": "providersne",
            "AssociatedResourceAccessibility": "Accessible",
            "DnsEntry": {
                "DnsName": "vpce-0da9c8491fcf3a6d9-snra-0eae7f31d03c50d72.rcfg-01ae20d51100803bd.4232ccc.vpc-lattice-rsc.us-east-1.on.aws",
                "HostedZoneId": "Z08285483B41F6ASEM5U1"
            },
            "AssociatedResourceArn": "arn:aws:vpc-lattice:us-east-1:661497052113:resourceconfiguration/rcfg-01ae20d51100803bd"
        },
        {
            "Id": "vpce-rsc-asc-0cb29006631b3cdca",
            "VpcEndpointId": "vpce-0da9c8491fcf3a6d9",
            "ServiceNetworkArn": "arn:aws:vpc-lattice:us-east-1:214517638895:servicenetwork/sn-06ce56215f97909e7",
            "ServiceNetworkName": "providersne",
            "AssociatedResourceAccessibility": "Accessible",
            "DnsEntry": {
                "DnsName": "vpce-0da9c8491fcf3a6d9-snra-012648bc4e8ad80dc.rcfg-0349a6320d67a56ad.4232ccc.vpc-lattice-rsc.us-east-1.on.aws",
                "HostedZoneId": "Z08285483B41F6ASEM5U1"
            },
            "AssociatedResourceArn": "arn:aws:vpc-lattice:us-east-1:001182731686:resourceconfiguration/rcfg-0349a6320d67a56ad"
        }
    ]
}
```

2. Log on to the EC2 instance in the SaaS provider VPC (for example with Session Manager) and initiate traffic using the DNS Name from the previous step. You should get a successful connection as in the example below. If you remember, we are using a sample application listening on TCP 1234 to simulate a resource in customer VPC.

```bash
sh-4.2$ telnet vpce-0da9c8491fcf3a6d9-snra-0eae7f31d03c50d72.rcfg-01ae20d51100803bd.4232ccc.vpc-lattice-rsc.us-east-1.on.aws 1234
Trying 10.0.1.178...
Connected to vpce-0da9c8491fcf3a6d9-snra-0eae7f31d03c50d72.rcfg-01ae20d51100803bd.4232ccc.vpc-lattice-rsc.us-east-1.on.aws.
Escape character is '^]'.
Connection successful!
Connection closed by foreign host.
sh-4.2$
```

Next, connect to the second customer instance in the same manner. This concludes the walkthrough. You can enhance this solution further by automating the steps shown in this blog and using a Private Hosted Zone to generate more user friendly hostnames, instead of the system-generated DNS names.

## Conclusion

Resource Gateways and VPC Lattice provide a powerful solution for SaaS providers to securely access customer resources in multi-tenant architectures. By leveraging these technologies, you can overcome traditional networking challenges, maintain strict security boundaries, and scale your connectivity to thousands of customers.

The architecture you've just built offers a flexible foundation that works well for most multi-tenant services. It enables you to securely access customer resources across account boundaries while simplifying network management and reducing operational complexity. The solution's ability to scale horizontally accommodates growing numbers of tenants, and its design maintains clear traffic separation between inbound and outbound flows.
