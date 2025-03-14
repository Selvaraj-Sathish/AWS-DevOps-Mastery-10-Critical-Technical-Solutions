# **EC2 Key Recovery: Regaining Access to Ubuntu Instances with Lost SSH Keys**

## ** Introduction**  
Losing an SSH private key can prevent you from accessing your EC2 instance. This guide provides step-by-step instructions to recover access to an **Ubuntu-based EC2 instance** without data loss.  

## ** Prerequisites**  
Before you start, ensure you have:  
- ✅ AWS Management Console access  
- ✅ IAM permissions for EC2 instance management  
- ✅ Basic knowledge of SSH, key pairs, and Ubuntu  
- ✅ The ability to stop and start EC2 instances  

---

## ** Methods to Recover Access**  
You can recover access using:  
- ✅ **Method 1: Create a New Key Pair and Replace the Old One** *(Recommended)*  
- ✅ **Method 2: Use a Recovery Instance to Modify the Authorized Keys**  
- ✅ **Method 3: Use AWS Systems Manager Session Manager (If Enabled)**  

---

## ** Method 1: Create a New Key Pair and Replace the Old One**  

### **Step 1: Create a New Key Pair in AWS**  
1. Navigate to **AWS EC2 Dashboard**  
2. Click **Key Pairs** → **Create Key Pair**  
3. Name it (e.g., `ubuntu-recovery-key`)  
4. Choose **RSA** and download the `.pem` file  

### **Step 2: Stop the Instance**  
1. In the EC2 Dashboard, find your locked instance  
2. Click **Instance State** → **Stop Instance**  

### **Step 3: Detach the Root Volume**  
1. Click **Volumes** in the EC2 Dashboard  
2. Locate the **root volume** attached to your instance  
3. Select the volume → Click **Actions** → **Detach Volume**  

### **Step 4: Attach the Volume to a Recovery Instance**  
1. **Launch a new temporary Ubuntu EC2 instance**  
2. Attach the **detached volume** to the new instance as `/dev/xvdf`  

### **Step 5: Modify the SSH Key on the Volume**  
 
````
   ssh -i ubuntu-recovery-key.pem ubuntu@<Recovery_Instance_IP>
````




### Mount the attached volume:

```
sudo mkdir /mnt/recovery
sudo mount /dev/xvdf1 /mnt/recovery
````
### Navigate to the .ssh folder of the original instance:

````
cd /mnt/recovery/home/ubuntu/.ssh
````

### Add the new public key to authorized_keys:

````
echo "ssh-rsa AAAA..." >> authorized_keys
````
## (Replace AAAA... with the public key from your new key pair.)

- Set proper permissions:

```
sudo chmod 600 authorized_keys
sudo chown ubuntu:ubuntu authorized_keys
```
- Unmount and detach the volume:

```
cd ~
sudo umount /mnt/recovery

```
### Step 6: Reattach the Volume to the Original Instance
- In the AWS Console, go to Volumes
- Select the modified volume → Attach Volume
- Attach it back to the original instance as /dev/xvda

### Step 7: Start the Instance and Connect
- Start the instance from Instance State → Start
- SSH into the instance using the new key:

```
ssh -i ubuntu-recovery-key.pem ubuntu@<Original_Instance_IP>
```
🎉 Success! You've regained access.

### Method 2: Use a Recovery Instance to Modify the Authorized Keys
If you have another Ubuntu instance with SSH access, you can use it to manually edit the authorized_keys file by mounting the lost instance’s root volume. The steps are the same as Method 1, but instead of generating a new key pair, you manually modify the existing SSH key file.

### Method 3: Use AWS Systems Manager Session Manager (If Enabled)
If AWS Systems Manager (SSM) Agent was enabled on your instance before losing access, you can regain access without SSH.

- Step 1: Enable SSM for the Instance
Navigate to EC2 Dashboard → Instances
Select the instance → Click Actions → Security → Modify IAM Role
Attach a role with AmazonSSMManagedInstanceCore permissions

- Step 2: Start a Session via Systems Manager
Open the AWS Systems Manager Console
Click Session Manager → Start Session
Select your instance and click Start Session
- Step 3: Modify the SSH Keys
Run the following command inside the session:
```
echo "ssh-rsa AAAA..." >> /home/ubuntu/.ssh/authorized_keys
```
Exit the session and try SSH access with your new key
✅ Done! This method avoids stopping and detaching the instance.

### Best Practices to Avoid Future Key Loss
- Use IAM User Roles: Reduce dependency on SSH keys
- Enable AWS Systems Manager: Provides an alternative access method
- Backup Private Keys: Store them securely in a password manager
- Use Multi-Factor Authentication (MFA): Enhances AWS Console security
- Create Multiple Key Pairs: Assign secondary keys to critical instances

### Summary
Losing your EC2 SSH key on Ubuntu is recoverable. In this guide, we covered:

✅ Creating and using a new key pair
✅ Modifying authorized_keys via a recovery instance
✅ Accessing an instance using AWS Systems Manager
