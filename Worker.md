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
