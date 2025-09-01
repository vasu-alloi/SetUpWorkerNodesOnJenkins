### Step 1: Prepare a Base Worker Node

#### 1.Launch a new EC2 instance (Ubuntu preferred).

Use the same VPC & Subnet where your Jenkins Master can reach it.

#### Assign IAM Role with policies for:

AmazonEC2FullAccess (for testing, later restrict).

CloudWatchAgentServerPolicy (optional, for monitoring).

Enable SSH key pair access.

#### Add Security Group rules to allow:

SSH (22) from Jenkins Master IP.

Jenkins Master → Worker Communication (default TCP 50000 for JNLP).

#### 2.Connect to the instance and install dependencies:
```
# Update & install packages
sudo apt update -y
sudo apt install -y openjdk-17-jdk git maven docker.io
sudo usermod -aG docker ubuntu

```
#### 3.Test connection with Jenkins Master:
```
ssh -i mykey.pem ubuntu@<worker-public-ip>
```
#### Step 2: Create an AMI

Once the worker is ready, stop unnecessary services and shut down.

From EC2 Console → Create Image (AMI) from the worker instance.

Name it: JenkinsWorkerBaseAMI.

This AMI will be the base for your ASG workers.

### Step 3: Create a Launch Template

Go to EC2 → Launch Templates → Create new template.

AMI: JenkinsWorkerBaseAMI.

Instance type: t3.medium (adjust as per workload).

Key pair: Same as used for SSH testing.

Security Group: Must allow Jenkins master to connect (50000, 22).

User Data (optional, auto-install Jenkins agent if you want):
```
#!/bin/bash
apt update -y
apt install -y openjdk-17-jdk
```
Save template.
### Step 4: Create Auto Scaling Group (ASG)

Go to EC2 → Auto Scaling Groups → Create ASG.

Choose Launch Template from Step 3.

Select same VPC & Subnet as Jenkins Master.

#### Scaling options:

Desired capacity: 0 (so no idle cost).

Min: 0.

Max: 5 (depends on expected load).

Use On-Demand or Spot Instances (cheaper but less reliable).

Finish setup.

Now you have an ASG that can spin up workers when Jenkins requests.

### Step 5: Configure Jenkins Master

Install required plugins in Jenkins Master:

EC2 Plugin or EC2 Fleet Plugin.

SSH Build Agents.

Add Worker Node Credentials:

Go to Jenkins → Manage Jenkins → Credentials → Add.

Type: SSH Username with private key.

Username: ubuntu (or AMI’s default user).

Paste private key (.pem file contents).

#### Configure Jenkins Cloud (if EC2 plugin):

Manage Jenkins → Nodes and Clouds → Configure Clouds.

Add Amazon EC2 cloud.

Provide AWS credentials (IAM User/Role).

Add AMI ID (your JenkinsWorkerBaseAMI).

Set remote user: ubuntu.

Connect via SSH.

Label workers (e.g., linux-ephemeral).

#### Test by launching a new worker:

Jenkins will request ASG → ASG spins up EC2 → Jenkins connects → Runs pipeline.

#### Step 6: Connection between Jenkins Master & Worker

Jenkins connects over SSH or JNLP (TCP/50000).

Make sure Security Groups allow this.

You can verify with:
```
ssh -i <key.pem> ubuntu@<worker-private-ip>
```
Once pipeline finishes, Jenkins (with EC2 plugin/ASG integration) can terminate the worker automatically.

🔹 Step 7: Running Pipelines

Example pipeline using ephemeral workers:

```
pipeline {
    agent {
        label 'workerNode' // give your agent name in placec of workerNode
    }
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t myapp .'
            }
        }
    }
}
```
Workers will spin up → run job → terminate after idle time.

Now you have a full setup: Base worker → AMI → Launch Template → ASG → Jenkins Integration → Ephemeral pipelines.
