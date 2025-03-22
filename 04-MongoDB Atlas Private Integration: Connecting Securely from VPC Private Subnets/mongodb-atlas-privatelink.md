# MongoDB Atlas Private Integration: Connecting Securely from VPC Private Subnets

## Chapter 1: Setting Up Private MongoDB Atlas Access from a VPC

### Overview

In this chapter, we'll establish a secure connection to MongoDB Atlas through AWS PrivateLink. This approach allows your application to connect to MongoDB without exposing traffic to the public internet, enhancing your security posture.

### Architecture Overview

We'll implement the following architecture:
- AWS VPC with both public and private subnets
- Ubuntu EC2 instance deployed in the private subnet
- MongoDB Atlas PrivateLink endpoint for secure database access
- Bastion host in the public subnet for SSH access

### Prerequisites

- An active MongoDB Atlas account with an M10+ cluster (required for PrivateLink)
- AWS account with administrative access
- AWS CLI installed and configured (optional)
- Basic understanding of VPC networking concepts

### Implementation Steps

#### Step 1: Create a VPC with Public and Private Subnets

1. **Navigate to VPC Dashboard**
   - Log into AWS Management Console
   - Search for "VPC" and select the VPC service

2. **Create a new VPC**
   - Click "Create VPC"
   - Select "VPC and more" to create a VPC with subnets
   - Enter a name tag (e.g., "MongoDB-VPC")
   - Enter IPv4 CIDR block (e.g., 10.0.0.0/16)
   - Number of Availability Zones: 2
   - Number of public subnets: 2
   - Number of private subnets: 2
   - NAT gateways: 1 per AZ (to allow internet access from private subnets)
   - VPC endpoints: None (we'll create the MongoDB endpoint separately)
   - Click "Create VPC"

   ```
   VPC "MongoDB-VPC" created successfully
   VPC ID: vpc-0123456789abcdef0
   CIDR: 10.0.0.0/16
   
   Public Subnets:
   - public-subnet-1: subnet-0abcd1234efgh5678 (10.0.0.0/24) in us-east-1a
   - public-subnet-2: subnet-0ijkl5678mnop9012 (10.0.1.0/24) in us-east-1b
   
   Private Subnets:
   - private-subnet-1: subnet-0qrst9012uvwx3456 (10.0.2.0/24) in us-east-1a
   - private-subnet-2: subnet-0yzab3456cdef7890 (10.0.3.0/24) in us-east-1b
   
   NAT Gateways:
   - nat-0123456789abcdef0 in public-subnet-1
   - nat-0fedcba9876543210 in public-subnet-2
   
   Internet Gateway:
   - igw-0123456789abcdef0
   ```

3. **Verify VPC Creation**
   - Confirm your VPC, subnets, route tables, and internet gateway are created
   - Note the subnet IDs for both public and private subnets

#### Step 2: Launch EC2 Instance in Private Subnet

1. **Navigate to EC2 Dashboard**
   - In AWS Management Console, go to EC2 service

2. **Launch an EC2 Instance**
   - Click "Launch Instance"
   - Name your instance (e.g., "MongoDB-Client")
   - Select an Ubuntu 22.04 LTS AMI
   - Choose an instance type (t2.micro is sufficient for testing)
   - Select or create a key pair (optional for Session Manager access)
   - Under "Network settings":
     * Select the VPC you created
     * Select one of your private subnets
     * Auto-assign public IP: Disable
     * Create a new security group with the following rules:
       - Allow HTTPS (port 443) outbound to 0.0.0.0/0 (for SSM Agent)
   - Expand "Advanced details":
     * IAM Instance Profile: Create or select a role with AmazonSSMManagedInstanceCore policy
     * User data (to install MongoDB client tools):
       ```bash
       #!/bin/bash
       apt update
       apt install -y mongodb-clients telnet
       ```
   - Click "Launch Instance"

   ```
   Instance "MongoDB-Client" launched successfully
   Instance ID: i-0123456789abcdef0
   Private IP: 10.0.2.42
   AMI: Ubuntu 22.04 LTS
   Instance Type: t2.micro
   VPC: MongoDB-VPC
   Subnet: private-subnet-1
   Security Group: sg-0123456789abcdef0 (MongoDB-Client-SG)
   ```

3. **(Optional) Launch a Bastion Host in Public Subnet**
   - This step is optional if you're using Session Manager for access
   - Click "Launch Instance" again
   - Name your instance (e.g., "MongoDB-Bastion")
   - Select an Ubuntu 22.04 LTS AMI
   - Choose t2.micro instance type
   - Select a key pair for SSH access
   - Under "Network settings":
     * Select the VPC you created
     * Select one of your public subnets
     * Auto-assign public IP: Enable
     * Create a new security group with the following rules:
       - Allow SSH (port 22) from your IP address only
   - Click "Launch Instance"
   - Note the public IP address of your bastion host

   ```
   Instance "MongoDB-Bastion" launched successfully
   Instance ID: i-0fedcba9876543210
   Public IP: 54.123.45.67
   Private IP: 10.0.0.123
   AMI: Ubuntu 22.04 LTS
   Instance Type: t2.micro
   VPC: MongoDB-VPC
   Subnet: public-subnet-1
   Security Group: sg-0fedcba9876543210 (MongoDB-Bastion-SG)
   ```

#### Step 3: Configure MongoDB Atlas for PrivateLink

1. **Log in to MongoDB Atlas**
   - Navigate to your MongoDB Atlas dashboard at [cloud.mongodb.com](https://cloud.mongodb.com)

2. **Create or Select a Cluster**
   - Ensure you have an M10+ cluster (required for PrivateLink)
   - If creating a new cluster, select the same AWS region as your VPC

   ```
   Cluster "Udemy-Cluster" created successfully
   Tier: M10
   Region: us-east-1
   MongoDB Version: 5.0
   ```

3. **Access Network Settings**
   - Select your project
   - Navigate to Network Access in the left sidebar menu
   - Select the "Private Endpoint" tab

4. **Enable PrivateLink**
   - Click "Add Private Endpoint"
   - Select AWS as your cloud provider
   - Choose the AWS region that matches your VPC
   - Provide a descriptive name for your endpoint (e.g., "udemy-vpc-endpoint")
   - Click "Continue"

   ```
   Private Endpoint request initiated
   Cloud Provider: AWS
   Region: us-east-1
   Endpoint Name: udemy-vpc-endpoint
   Status: INITIATING
   ```

5. **Copy Generated Endpoint Service Name**
   - MongoDB Atlas will generate a service name

   ```
   Service Name: com.amazonaws.vpce.us-east-1.vpce-svc-0abc123def456789g
   ```

#### Step 4: Create VPC Endpoint in AWS Console

1. **Navigate to VPC Dashboard**
   - Open the AWS Management Console
   - Go to VPC service

2. **Create Endpoint**
   - In the left navigation pane, click "Endpoints"
   - Click "Create Endpoint"
   - Select "Find service by name"
   - Paste the MongoDB Atlas service name copied earlier
   - Click "Verify"

3. **Configure VPC Settings**
   - Select your VPC from the dropdown
   - Select the private subnets where your EC2 instance runs
   - Under "Security groups", create a new security group with the following rules:
     * Inbound: Allow TCP on ports 27015-27017 from your private subnet CIDR
     * Outbound: Allow TCP on ports 27015-27017 to 0.0.0.0/0

   ```
   Security Group "MongoDB-Endpoint-SG" created
   Security Group ID: sg-01234abcde5678fgh
   
   Inbound Rules:
   - TCP 27015-27017 from 10.0.2.0/24 (private-subnet-1)
   - TCP 27015-27017 from 10.0.3.0/24 (private-subnet-2)
   
   Outbound Rules:
   - TCP 27015-27017 to 0.0.0.0/0
   ```

4. **Create the Endpoint**
   - Review your settings
   - Click "Create endpoint"

   ```
   VPC Endpoint creation initiated
   Endpoint ID: vpce-0123456abcdef7890
   Service Name: com.amazonaws.vpce.us-east-1.vpce-svc-0abc123def456789g
   VPC: vpc-0123456789abcdef0 (MongoDB-VPC)
   Subnets: private-subnet-1, private-subnet-2
   Security Group: sg-01234abcde5678fgh (MongoDB-Endpoint-SG)
   Status: pending
   ```

#### Step 5: Finalize MongoDB Atlas Configuration

1. **Return to MongoDB Atlas**
   - Navigate back to the Private Endpoint tab in MongoDB Atlas

2. **Update Endpoint Status**
   - Find your pending endpoint
   - Click "Complete Setup"
   - Copy your AWS VPC Endpoint ID (format: `vpce-0123456abcdef7890`)
   - Paste it into the Atlas interface
   - Click "Save"

   ```
   Private Endpoint setup in progress
   Endpoint ID: vpce-0123456abcdef7890
   Status: WAITING_FOR_USER
   ```

3. **Wait for Endpoint Validation**
   - Atlas will verify the connection

   ```
   Private Endpoint setup completed
   Endpoint ID: vpce-0123456abcdef7890
   Status: AVAILABLE
   ```

4. **Get Private Connection String**
   - In MongoDB Atlas, navigate to your cluster
   - Click "Connect"
   - Select "Connect your application"
   - Copy the private connection string

   ```
   Connection String:
   mongodb+srv://username:password@udemy-cluster.abc123.mongodb.net/myDatabase?retryWrites=true&w=majority
   
   Hostname: udemy-cluster.abc123.mongodb.net
   ```

#### Step 6: Test the Connection from Private EC2 Instance

1. **Access Private EC2 Instance through AWS Console**
   - Navigate to the EC2 Dashboard in AWS Management Console
   - Select your instance in the private subnet (MongoDB-Client)
   - Click the "Connect" button at the top of the page
   - Choose "Session Manager" as the connection method
   - Click "Connect"

   ```
   Starting session with SessionId: admin-01234567890abcdef
   
   Welcome to Ubuntu 22.04.2 LTS (GNU/Linux 5.15.0-1031-aws x86_64)
   
   * Documentation:  https://help.ubuntu.com
   * Management:     https://landscape.canonical.com
   * Support:        https://ubuntu.com/advantage
   
   System information as of Wed Mar 22 12:36:42 UTC 2025
   
   ubuntu@ip-10-0-2-42:~$
   ```

   > Note: To use Session Manager, you need to have the AWS Systems Manager Agent (SSM Agent) installed on your EC2 instance and appropriate IAM permissions. This comes pre-installed on many Amazon AMIs, including recent Ubuntu AMIs.

3. **Test Connection with Telnet**
   - On the private EC2 instance, use telnet to test connectivity to MongoDB
   ```bash
   telnet udemy-cluster.abc123.mongodb.net 27017
   ```

   ```
   Trying 10.0.4.213...
   Connected to udemy-cluster.abc123.mongodb.net.
   Escape character is '^]'.
   
   ```

4. **Test with MongoDB Shell (Optional)**
   - If you installed MongoDB tools in user data, you can test with the mongo shell
   ```bash
   mongosh "mongodb+srv://udemy-cluster.abc123.mongodb.net" --username your-username
   ```

   ```
   Enter password: ************
   Current Mongosh Log ID: 65ac327e687f23c4a1b9d10a
   Connecting to: mongodb+srv://udemy-cluster.abc123.mongodb.net/test?appName=mongosh+1.10.1
   Using MongoDB: 5.0.16
   Using Mongosh: 1.10.1
   
   Atlas atlas-abcde1-shard-0 [primary] test>
   ```

### Troubleshooting Guide

1. **VPC Endpoint Issues**
   - Verify endpoint is in "Available" state in AWS console
   - Check that the endpoint ID matches what's in MongoDB Atlas
   - Confirm security group allows traffic on ports 27015-27017

2. **Connection Failures**
   - Check route tables for private subnets
   - Verify DNS resolution is working correctly
   ```bash
   nslookup udemy-cluster.abc123.mongodb.net
   ```
   - Ensure the MongoDB Atlas cluster is running
   - Verify that the MongoDB user has appropriate permissions

3. **Session Manager Connection Issues**
   - Verify that your instance has the AmazonSSMManagedInstanceCore policy attached
   - Check that your instance has outbound internet access (via NAT Gateway)
   - Ensure the SSM Agent is running on your instance: `sudo systemctl status amazon-ssm-agent`
   - Verify you have the necessary IAM permissions to use Session Manager

### Validation Checklist

- [ ] VPC created with public and private subnets
- [ ] EC2 instance launched in private subnet with SSM access
- [ ] MongoDB Atlas PrivateLink endpoint created and available
- [ ] AWS VPC endpoint created and available
- [ ] Successfully connected to private instance via Session Manager
- [ ] Telnet connection to MongoDB endpoint successful
- [ ] MongoDB shell connection successful (if tested)

### Additional Resources

- [MongoDB Atlas PrivateLink Documentation](https://docs.atlas.mongodb.com/security-private-endpoint/)
- [AWS PrivateLink Documentation](https://docs.aws.amazon.com/vpc/latest/privatelink/privatelink-share-your-services.html)
- [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)

This completes Chapter 1 of our Udemy course, establishing the foundation for secure MongoDB Atlas connections through private networking.


