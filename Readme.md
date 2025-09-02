### Jenkins Dynamic Worker Setup with AWS Auto Scaling
#### Overview
This guide walks you through setting up dynamic Jenkins worker nodes using AWS EC2, Auto Scaling Groups (ASG), and Launch Templates. This setup allows Jenkins to automatically spin up workers when needed and terminate them when idle, optimizing costs.
#### Architecture Flow
```
Jenkins Master → Triggers Build → ASG Spins Up Worker → Worker Connects via SSH/JNLP → Executes Job → Worker Terminates
```
#### Step 1: Prepare Base Worker Node
##### 1.1 Launch EC2 Instance

**OS:** Ubuntu (preferred)
**VPC/Subnet:** Same as Jenkins Master for connectivity
**IAM Role:** Attach policies for:
```
AmazonEC2FullAccess (restrict in production)
CloudWatchAgentServerPolicy (optional, for monitoring)
```


##### 1.2 SSH Key Setup - Two Methods
**Method 1:** AWS Key Pair (Recommended for AWS)

Go to AWS Console → EC2 → Key Pairs → Create key pair
**Settings:**

**Name:** jenkins-workers-key
**Type: **RSA
**Format:** PEM


Download the .pem file (keep secure!)
Launch worker with this key pair
