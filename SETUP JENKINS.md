### Step 1: Prepare a Base Worker Node

#### 1.Launch a new EC2 instance (Ubuntu preferred).

Use the same VPC & Subnet where your Jenkins Master can reach it.

<img width="1684" height="256" alt="image" src="https://github.com/user-attachments/assets/a902a061-97de-4af4-be7c-b88f738986a9" />

#### Assign IAM Role with policies for:

AmazonEC2FullAccess (for testing, later restrict).

CloudWatchAgentServerPolicy (optional, for monitoring).

Enable SSH key pair access.
#### Method1:

You need an EC2 Key Pair for Jenkins Master to connect to workers via SSH.

Steps:
```
1.In AWS Console → EC2 → Key Pairs → Create key pair.

2.Name: jenkins-workers-key.

3.Type: RSA.

4.Format: PEM.

5.Download the .pem file (keep safe!).

6.Launch your worker instance with this key pair.

7.In EC2 Launch Wizard → Key pair (login) → select jenkins-workers-key.
```
Test SSH:
```
chmod 400 jenkins-workers-key.pem
ssh -i jenkins-workers-key.pem ubuntu@<worker-public-ip>

```
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
```
a.Once the worker is ready, stop unnecessary services and shut down.

b.From EC2 Console → Create Image (AMI) from the worker instance.

c.Name it: JenkinsImage.
```
<img width="1694" height="184" alt="image" src="https://github.com/user-attachments/assets/365fb2fe-9ad7-4ee3-99eb-e45921e9ca8a" />

This AMI will be the base for your ASG workers.

### Step 3: Create a Launch Template
```
1.Go to EC2 → Launch Templates → Create new template.

2.AMI: JenkinsImage.

3.Instance type: t3.medium (adjust as per workload).

4.Key pair: Same as used for SSH testing.

5.Security Group: Must allow Jenkins master to connect (50000, 22).

6.User Data (optional, auto-install Jenkins agent if you want):
```
```
#!/bin/bash
apt update -y
apt install -y openjdk-17-jdk
```
7.Save template.
<img width="1670" height="159" alt="image" src="https://github.com/user-attachments/assets/e7d0f902-ac03-4542-99b5-f2258c7d61cc" />

### Step 4: Create Auto Scaling Group (ASG)
```
1.Go to EC2 → Auto Scaling Groups → Create ASG.

2.Choose Launch Template from Step 3.

3.Select same VPC & Subnet as Jenkins Master.

#### Scaling options:

4.Desired capacity: 0 (so no idle cost).

5.Min: 0.

6.Max: 5 (depends on expected load).

7.Use On-Demand or Spot Instances (cheaper but less reliable).

8.Finish setup.
```
<img width="1852" height="770" alt="image" src="https://github.com/user-attachments/assets/d4e0759a-4166-4bd8-b7f9-00a96b9a0984" />


Now you have an ASG that can spin up workers when Jenkins requests.

### Step 5: Configure Jenkins Master
```
1.Install required plugins in Jenkins Master:

2.EC2 Plugin or EC2 Fleet Plugin.

3.SSH Build Agents.

4.Add Worker Node Credentials:

5.Go to Jenkins → Manage Jenkins → Credentials → Add.

6.Type: SSH Username with private key.

7.Username: ubuntu (or AMI’s default user).

8.Paste private key (.pem file contents).
```

#### Configure Jenkins Cloud (if EC2 plugin):
```
1.Manage Jenkins → Nodes and Clouds → Configure Clouds.

2.Add Amazon EC2 cloud.

3.Provide AWS credentials (IAM User/Role).

4.Add AMI ID (your JenkinsWorkerBaseAMI).

5.Set remote user: ubuntu.

6.Connect via SSH.

7.Label workers (e.g., workerNode).
```
#### Test by launching a new worker:

Jenkins will request ASG → ASG spins up EC2 → Jenkins connects → Runs pipeline.

#### (Alternative): Configure Jenkins Master with EC2-Fleet Plugin
```
1. Install Plugin

Go to Manage Jenkins → Plugins → Available plugins.

Search for EC2 Fleet Plugin → Install.

2. Add AWS Credentials to Jenkins

Go to Manage Jenkins → Credentials → Add.

Kind: AWS Credentials.

ID: aws-jenkins.

Enter Access Key & Secret Key (or rely on IAM Role if Jenkins Master runs on EC2 with proper permissions).

Add another credential for SSH Private Key:

Kind: SSH Username with private key.

Username: ubuntu.

Paste .pem private key contents.

3. Configure EC2 Fleet Cloud

Go to Manage Jenkins → Nodes and Clouds → Configure Clouds.

Add new cloud → EC2 Fleet.

Fill in details:

Name: aws-fleet.

AWS Credentials: select aws-jenkins.

Region: your AWS region (e.g., ap-south-1).

Fleet ID:

If you already created an Auto Scaling Group (ASG) → use the ASG ARN/Name.

If you’re using an EC2 Spot Fleet/Capacity Reservation → use Fleet ID.
```
#### Step 6: Connection between Jenkins Master & Worker

Jenkins connects over SSH or JNLP (TCP/50000).

Make sure Security Groups allow this.

You can verify with:
```
ssh -i <key.pem> ubuntu@<worker-private-ip>
```
Once pipeline finishes, Jenkins (with EC2 plugin/ASG integration) can terminate the worker automatically.

### Step 7: Running Pipelines

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
