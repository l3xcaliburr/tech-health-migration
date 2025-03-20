# AWS Migration Project

This project uses AWS CDK with TypeScript to set up a secure cloud infrastructure for application migration. It creates a VPC with public and private subnets across multiple availability zones, deploys an EC2 instance, and provisions a MySQL RDS database.

## Architecture

- **VPC**: 2 Availability Zones with public and private subnets
- **EC2 Instance**: Deployed in public subnet with security group for SSH and HTTP access
- **RDS Database**: MySQL 8.0 deployed in private subnet
- **Security Groups**: Properly configured to allow only necessary traffic
- **IAM Roles**: Set up for EC2 instance with appropriate permissions

## Prerequisites

- AWS CLI installed and configured
- Node.js (v14 or higher)
- AWS CDK installed globally

## Setup Instructions

1. **Clone the repository**

   ```bash
   git clone <repository-url>
   cd aws-migration-project
   ```

2. **Install dependencies**

   ```bash
   npm install
   ```

3. **Bootstrap your AWS environment (if not already done)**

   ```bash
   cdk bootstrap
   ```

4. **Deploy the stack**
   ```bash
   cdk deploy
   ```

## Connecting to Resources

### SSH into EC2 Instance

```bash
ssh -i "your-key-pair.pem" ec2-user@<ec2-public-ip>
```

### MySQL Installation on EC2

For Amazon Linux 2023:

```bash
sudo dnf install mariadb105
```

For Amazon Linux 2:

```bash
sudo yum install mysql -y
```

### Connect to RDS from EC2

```bash
mysql -h <rds-endpoint> -u admin -p
```

## Key Components

### VPC Configuration

```typescript
const vpc = new ec2.Vpc(this, "MigrationVPC", {
  maxAzs: 2,
  subnetConfiguration: [
    {
      cidrMask: 24,
      name: "PublicSubnet",
      subnetType: ec2.SubnetType.PUBLIC,
    },
    {
      cidrMask: 24,
      name: "PrivateSubnet",
      subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
    },
  ],
});
```

### EC2 Instance

```typescript
const ec2Instance = new ec2.Instance(this, "MigrationEC2", {
  vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC },
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.T3,
    ec2.InstanceSize.MICRO
  ),
  machineImage: ec2.MachineImage.latestAmazonLinux(),
  securityGroup: ec2SecurityGroup,
  role: ec2Role,
});
```

### RDS Instance

```typescript
new rds.DatabaseInstance(this, "MigrationRDS", {
  engine: rds.DatabaseInstanceEngine.mysql({
    version: rds.MysqlEngineVersion.VER_8_0,
  }),
  vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
  securityGroups: [rdsSecurityGroup],
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.BURSTABLE3,
    ec2.InstanceSize.MICRO
  ),
  allocatedStorage: 20,
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

## Troubleshooting

### Common Issues

- **RDS Instance Type Compatibility**: Make sure to use BURSTABLE3 (t3) instances with MySQL 8.0, as t2 instances aren't compatible.
- **Connection Issues**: Verify that security groups are correctly configured to allow traffic between EC2 and RDS.
- **Bootstrap Issues**: If deployment fails with bootstrap errors, run `cdk bootstrap` first.

## Cleanup

To avoid ongoing charges, destroy the stack when no longer needed:

```bash
cdk destroy
```

**Note**: By default, the RDS instance has `RemovalPolicy.DESTROY`, which means the database will be deleted when the stack is destroyed. Change this setting if you need to retain your data.

## Security Considerations

- Database credentials are automatically stored in AWS Secrets Manager
- EC2 instance is configured with IAM role for secure access to AWS resources
- Security groups are configured to allow only necessary traffic
