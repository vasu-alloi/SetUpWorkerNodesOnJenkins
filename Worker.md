### Ephemeral Workers on AWS EC2 (recommended for scalability)

This way workers spin up only when needed.

### Steps:

### Install EC2 Plugin in Jenkins:

Manage Jenkins → Plugin Manager → Available → EC2 Plugin.

### Configure EC2 cloud:

Manage Jenkins → Nodes → Clouds → Add a new cloud → Amazon EC2.

Provide AWS credentials + region.

Define an AMI with Java + build tools (or use Amazon Linux/Ubuntu and init script).

### Configure templates:

Set instance type (e.g., t3.medium).

Remote root directory = /home/jenkins.

Labels = ec2-agent.

Launch method = SSH (Jenkins will inject keys).

Test:

Run a pipeline with agent { label 'ec2-agent' }.

Jenkins will spin up EC2 → run job → terminate after use.


### 1. Jenkins Master Node

Upgrade Instance Type → If your Jenkins master is slow (UI lag, queue delays, config saves taking long), increasing instance type (e.g., t3.medium → t3.large or even m5.large) helps.

Storage → Move /var/lib/jenkins to gp3 / io2 EBS for faster I/O.

Reserved Instance → Since Jenkins master should be always running, better to use Reserved Instance (RI) or Savings Plan.

On-Demand = flexibility, higher cost.

RI = predictable workload, cheaper (30–70% savings).

✅ Recommendation: Keep master on On-Demand or Reserved (not Spot), since you don’t want your control plane to die.

### 🔹 2. Jenkins Worker Nodes

### Here’s the big decision point:

Option A: Static Worker Nodes

You provision EC2 worker nodes permanently.

Simple, but costly (workers may sit idle).

Option B: Ephemeral Worker Nodes (Recommended)

Use EC2 Auto Scaling Group (with Spot + On-Demand mix).

Jenkins EC2 plugin launches workers only when jobs are queued.

Worker terminates after job → saves cost, improves scalability.

✅ Recommendation: Ephemeral worker nodes via ASG (EC2 plugin).
This ensures:

Jobs don’t hog the master.

You scale on-demand.

Cheaper (use Spot workers safely).

### 🔹 3. Performance Tuning

Plugins → Remove unused plugins (too many = slow Jenkins).

JVM Tuning → Add flags like -Xms1G -Xmx4G (depending on master size).

Garbage Collection → Use G1GC for smoother memory handling.

Logs & Builds Cleanup → Enable automatic log rotation & job history cleanup.

Storage → Ensure builds/artifacts are on S3 or external storage, not filling /var/lib/jenkins.

🎯 My Recommendation (as your Jenkins Manager)

Master: Run on m5.large or c5.large On-Demand/Reserved (never Spot).

Workers: Run ephemeral workers using ASG + Spot instances (saves cost, scalable).

Performance: Audit plugins, tune JVM, and move artifacts/logs to S3.

👉 This way, your master is stable (always up), while your workers are elastic & cost-efficient.
