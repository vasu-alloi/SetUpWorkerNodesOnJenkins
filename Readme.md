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
```
1.Stop unnecessary services on the worker
2.Clean up temporary files
3.Shutdown the instance
```
**2.2 Create AMI**
```
1.Go to EC2 Console → Select your worker instance
2.Actions → Image and templates → Create image
3.Name: JenkinsWorkerBaseAMI
4.Description: "Base image for Jenkins workers with Java, Docker, Maven"
5.Click Create Image
```

Note: This AMI becomes the template for all future worker instances


#### Step 3: Create Launch Template
**3.1 Launch Template Configuration**
```
1.Go to EC2 → Launch Templates → Create launch template
2.Template name: jenkins-worker-template
3.AMI: Select your JenkinsWorkerBaseAMI
4.Instance type: t3.medium (adjust based on workload)
5.Key pair: jenkins-workers-key (from Step 1)
6.Security groups: Same as configured in Step 1.3
```
**3.2 User Data (Optional Auto-Setup)**
```
#!/bin/bash
apt update -y
apt install -y openjdk-17-jdk
# Additional setup commands if needed
```
**3.3 Save Template**
Click Create launch template

#### Step 4: Create Auto Scaling Group (ASG)
**4.1 ASG Configuration**
```
1.Go to EC2 → Auto Scaling Groups → Create Auto Scaling group
2.Name: jenkins-workers-asg
3.Launch template: Select jenkins-worker-template
4.VPC: Same as Jenkins Master
5.Subnets: Same availability zones as Jenkins Master
```
**4.2 Scaling Settings**
```
Desired capacity: 0 (no idle costs)
Minimum capacity: 0
Maximum capacity: 5 (adjust based on expected load)
Instance type: On-Demand or Spot (Spot is cheaper but less reliable)
```
**4.3 Finish Setup**
Complete ASG creation. Workers will now spin up on-demand.

#### Step 5: Configure Jenkins Master
**5.1 Install Required Plugins**
```
1.Go to Manage Jenkins → Plugins → Available plugins:

2.EC2 Plugin or EC2 Fleet Plugin
3.SSH Build Agents Plugin
```
**5.2 Add SSH Credentials**

1.Manage Jenkins → Credentials → Add Credentials
2.Kind: SSH Username with private key
3.Username: ubuntu (or your AMI's default user)
4.Private Key: Paste contents of your .pem file or private key
```
cat ~/.ssh/id_rsa  # Copy this content
```
**ID: jenkins-worker-ssh-key**

**5.3 Configure Cloud (EC2 Plugin Method)**

    1.Manage Jenkins → Nodes and Clouds → Configure Clouds
    2.Add new cloud → Amazon EC2
    3.Configuration:
```
Name: aws-workers
AWS Credentials: Add AWS access credentials
Region: Your AWS region (e.g., us-east-1)
EC2 Key Pair's Private Key: Select SSH credential from Step 5.2
AMI ID: Your JenkinsWorkerBaseAMI ID
Remote user: ubuntu
Labels: worker-node (for pipeline targeting)
```


**5.4 Alternative: EC2 Fleet Plugin Configuration**

1.Manage Jenkins → Nodes and Clouds → Configure Clouds
2.Add new cloud → EC2 Fleet
3.Settings:
```
Name: aws-fleet
AWS Credentials: Your AWS credentials
Region: Your AWS region
Fleet ID: Your ASG ARN/Name
SSH Credentials: Select from Step 5.2
```



#### Step 6: Test Connection & Verification
**6.1 Manual Connection Test**
**From Jenkins Master:**
```
# Test SSH connection to worker
ssh -i jenkins-workers-key.pem ubuntu@<worker-private-ip>

# Verify Java installation
java -version

# Check Docker access
docker --version
sudo docker ps
```
**6.2 Jenkins Connection Test**
```
1.Manage Jenkins → Nodes and Clouds → New Node
2.Node name: test-worker
3.Type: Permanent Agent
4.Launch method: Launch agents via SSH
5.Host: Worker instance IP
6.Credentials: Select your SSH credential
7.Save and check connection logs
```

#### Step 7: Create and Run Pipelines
**7.1 Example Pipeline Using Dynamic Workers**
```
pipeline {
    agent {
        label 'worker-node'  // Matches the label from cloud config
    }
    
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-repo/your-project.git'
            }
        }
        
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        
        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t myapp:${BUILD_NUMBER} .
                    docker tag myapp:${BUILD_NUMBER} myapp:latest
                '''
            }
        }
        
        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }
    }
    
    post {
        always {
            cleanWs() // Clean workspace after build
        }
    }
}
```

**7.2 Pipeline Execution Flow**
```
1.Pipeline triggered → Jenkins requests worker
2.ASG spins up new EC2 instance from AMI
3.Worker connects to Jenkins via SSH/JNLP
4.Job executes on the worker
5.Worker terminates after idle time (cost optimization)
```

### Step 8: Security Considerations
**8.1 Network Security**
```
VPC: Use private subnets for workers when possible
Security Groups: Restrict access to minimum required ports
NACLs: Additional network-level security if needed
```
**8.2 IAM Best Practices**
```
Principle of least privilege: Restrict IAM policies in production
Instance profiles: Use IAM roles instead of access keys when possible
Rotate keys: Regular rotation of SSH keys and AWS credentials
```
**8.3 Jenkins Security**
```
Secure credentials storage: Use Jenkins credential store
Node security: Configure node security appropriately
Build isolation: Ensure builds don't interfere with each other
```
#### Troubleshooting Common Issues
**Connection Problems**
```
# Check Jenkins Master → Worker connectivity
telnet <worker-ip> 22
telnet <worker-ip> 50000

# Verify SSH key permissions
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub

# Check Jenkins logs
tail -f /var/log/jenkins/jenkins.log
```
**ASG Issues**
```
Check Launch Template: Verify AMI, security groups, key pairs
IAM Permissions: Ensure Jenkins has EC2 permissions
VPC Connectivity: Verify network routing between master and workers
```
**Performance Optimization**
```
Instance types: Choose appropriate instance sizes for workload
Spot instances: Use for cost savings (with interruption handling)
Monitoring: Set up CloudWatch for worker metrics
```

**Cost Optimization Tips**
```
1.Use Spot Instances: Up to 90% cost savings
2.Right-size instances: Match instance type to workload
3.Auto-termination: Configure idle timeout for workers
4.Scheduled scaling: Scale down during off-hours
4.Reserved instances: For predictable baseline capacity
```

**Key Benefits of This Setup**
```
Cost Efficient: Pay only for compute time used
Scalable: Automatically handles varying workloads
Isolated: Each build runs on fresh environment
Reliable: Auto-healing through ASG
Flexible: Easy to modify worker configurations
```

**Quick Reference Commands**
```
# Generate SSH key
ssh-keygen -t ed25519 -C "jenkins@company.com"

# Test SSH connection
ssh -i jenkins-key.pem ubuntu@<worker-ip>

# Check Java on worker
java -version

# Verify Docker access
docker ps

# Monitor Jenkins logs
sudo tail -f /var/log/jenkins/jenkins.log
```
This setup provides a robust, scalable, and cost-effective Jenkins infrastructure that automatically manages worker lifecycle based on build demand.
