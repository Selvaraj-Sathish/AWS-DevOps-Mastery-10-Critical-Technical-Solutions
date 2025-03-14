# Secure S3 Access Through EC2 IAM Roles: No Credentials Needed

## Overview  
This guide explains how to securely grant an EC2 instance access to an S3 bucket using IAM roles, eliminating the need for static credentials. It also covers testing the configuration and monitoring access via AWS CloudTrail.  

---

## 1. Create an IAM Role for EC2  

### Step 1: Navigate to the AWS IAM Console  
- Open the **IAM** service from the AWS Management Console.  
- Click on **Roles** in the left-hand menu.  
- Click **Create Role**.  

### Step 2: Select the Trusted Entity  
- Choose **AWS Service** as the trusted entity.  
- Select **EC2** as the service using this role.  
- Click **Next**.  

### Step 3: Attach Permissions Policy  
- Search for and attach the **AmazonS3ReadOnlyAccess** policy (or create a custom policy with specific S3 permissions).  
- Click **Next**.  

### Step 4: Set Role Name and Create Role  
- Enter a meaningful role name (e.g., `EC2-S3-Access-Role`).  
- Click **Create Role**.  

---

## 2. Attach the IAM Role to an EC2 Instance  

### Step 1: Navigate to the EC2 Console  
- Open the **EC2** service in AWS.  
- Select the instance that needs S3 access.  

### Step 2: Modify IAM Role  
- Click on **Actions** → **Security** → **Modify IAM Role**.  
- Choose the IAM role you created (`EC2-S3-Access-Role`).  
- Click **Update IAM Role**.  

---

## 3. Test S3 Access from EC2  

### Step 1: Connect to the EC2 Instance  
- Use SSH to connect to your instance:  
  ```bash
  ssh -i your-key.pem ec2-user@your-ec2-public-ip
  ```  

### Step 2: Verify IAM Role Association  
- Run the following command to check if the instance has the correct role:  
  ```bash
  curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
  ```  

### Step 3: Test S3 Access  
- Run the following AWS CLI command to list S3 buckets:  
  ```bash
  aws s3 ls
  ```  
- To list objects in a specific bucket:  
  ```bash
  aws s3 ls s3://your-bucket-name/
  ```  
- To download a file from S3:  
  ```bash
  aws s3 cp s3://your-bucket-name/file.txt .
  ```  

If the commands execute successfully without requiring credentials, the IAM role is correctly set up.  

---

## 4. Monitor S3 Access with AWS CloudTrail  

### Step 1: Enable CloudTrail Logging  
- Navigate to the **AWS CloudTrail** console.  
- Click **Create trail**.  
- Provide a name (e.g., `S3-Access-Trail`).  
- Under **Management Events**, enable **Read and Write** for S3.  
- Choose an S3 bucket to store the logs.  

### Step 2: Analyze S3 Access Logs  
- Go to **CloudTrail Logs** in the chosen S3 bucket.  
- Look for events with `s3.amazonaws.com` in the event source.  
- Use the following AWS CLI command to search logs:  
  ```bash
  aws s3 cp s3://your-cloudtrail-bucket/AWSLogs/your-account-id/CloudTrail/your-log-file.json .
  cat your-log-file.json | grep S3
  ```  

---

## Conclusion  
By implementing **EC2 instance profiles**, we enable secure, credential-free access to S3. We tested access using AWS CLI and ensured proper monitoring with CloudTrail logs. This approach improves security by eliminating static credentials and enforcing IAM role-based access.  

