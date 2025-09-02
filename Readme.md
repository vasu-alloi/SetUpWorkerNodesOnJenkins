### Jenkins Dynamic Worker Setup with AWS Auto Scaling
#### Overview
This guide walks you through setting up dynamic Jenkins worker nodes using AWS EC2, Auto Scaling Groups (ASG), and Launch Templates. This setup allows Jenkins to automatically spin up workers when needed and terminate them when idle, optimizing costs.
#### Architecture Flow
```
Jenkins Master → Triggers Build → ASG Spins Up Worker → Worker Connects via SSH/JNLP → Executes Job → Worker Terminates
```
#### Step 1: Prepare Base Worker Node
#### 1.1 Launch EC2 Instance

**OS:** Ubuntu (preferred)
**VPC/Subnet:** Same as Jenkins Master for connectivity
**IAM Role:** Attach policies for:
```
AmazonEC2FullAccess (restrict in production)
CloudWatchAgentServerPolicy (optional, for monitoring)
```


#### 1.2 SSH Key Setup - Two Methods
**Method 1:** AWS Key Pair (Recommended for AWS)

1.Go to AWS Console → EC2 → Key Pairs → Create key pair
2.**Settings:**

**Name:** jenkins-workers-key
**Type: **RSA
**Format:** PEM


**3.Download the .pem file (keep secure!)
4.Launch worker with this key pair
5.Test connection:**
```
chmod 400 jenkins-workers-key.pem
ssh -i jenkins-workers-key.pem ubuntu@<worker-public-ip>
```
**Method 2: Generate Custom SSH Keys**
**1.Generate key pair**
```
# RSA key (4096 bits)
ssh-keygen -t rsa -b 4096 -C "jenkins@your-domain.com"

# OR ED25519 key (more secure)
ssh-keygen -t ed25519 -C "jenkins@your-domain.com"
```
**2.Copy public key to worker:**
```
# Method A: Using ssh-copy-id
ssh-copy-id -i ~/.ssh/id_rsa.pub username@agent-machine-ip

# Method B: Manual copy
cat ~/.ssh/id_rsa.pub | ssh username@agent-machine-ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```
**3.Test connection:**
```
ssh -i ~/.ssh/id_rsa username@agent-machine-ip
```
**1.3 Configure Security Groups**
Allow these ports on worker security group:

SSH (22): From Jenkins Master IP
JNLP (50000): From Jenkins Master (for agent communication)

**1.4 Install Dependencies on Worker**
```
# Update system and install required packages
sudo apt update -y
sudo apt install -y openjdk-17-jdk git maven docker.io

# Add ubuntu user to docker group
sudo usermod -aG docker ubuntu

# Verify Java installation
java -version
```
#### Step 2: Create AMI (Amazon Machine Image)
**2.1 Prepare Instance for AMI**

1.Stop unnecessary services on the worker
2.Clean up temporary files
3.Shutdown the instance

**2.2 Create AMI**

1.Go to EC2 Console → Select your worker instance
2.Actions → Image and templates → Create image
3.Name: JenkinsWorkerBaseAMI
4.Description: "Base image for Jenkins workers with Java, Docker, Maven"
5.Click Create Image


Note: This AMI becomes the template for all future worker instances


#### Step 3: Create Launch Template
**3.1 Launch Template Configuration**

1.Go to EC2 → Launch Templates → Create launch template
2.Template name: jenkins-worker-template
3.AMI: Select your JenkinsWorkerBaseAMI
4.Instance type: t3.medium (adjust based on workload)
5.Key pair: jenkins-workers-key (from Step 1)
6.Security groups: Same as configured in Step 1.3

**3.2 User Data (Optional Auto-Setup)**
```
#!/bin/bash
apt update -y
apt install -y openjdk-17-jdk
# Additional setup commands if needed
```


