Key Highlights:
Provision AWS EC2 Instances:

Jenkins controller and agents are configured with specific AMIs, instance types, and security groups.
Prepare Servers:

Updates, Java installation, and optional hostname configuration are performed on all servers.
Install Jenkins on Controller:

Jenkins is installed and configured, with instructions to access the Jenkins UI and complete the setup wizard.
Configure SSH for Agent Connection:

SSH keys are generated on the controller and shared with agents for secure connectivity.
Add Agent Nodes in Jenkins UI:

Steps to configure Jenkins nodes via SSH for Java and Oracle agents are provided.
Install Build Tools on Agents:

Java agent is equipped with Git, Maven, and Gradle; Oracle agent requires specific Oracle tools for backend builds.
Jenkins Pipeline Jobs:

Example pipeline scripts for Java and Oracle builds are shared.
Connect Jenkins to GitHub:

Instructions for setting up GitHub plugins, credentials, and webhooks to trigger builds.
Testing and Validation:

Steps to validate the setup through GitHub code pushes and agent-specific job execution.
Optional Security Hardening:

Recommendations for securing Jenkins, including HTTPS, LDAP/SSO, and regular updates.


# Step-by-Step Jenkins Controller-Agent Setup on AWS

## 1. Provision AWS EC2 Instances

### a. Jenkins Controller (Master)
- **AMI:** Ubuntu Server 22.04 LTS (or latest stable)
- **Type:** t3.medium or higher
- **Security Group:** Allow TCP 8080 (Jenkins UI), 22 (SSH) from your IP

### b. Jenkins Agent 1 (Java)
- **AMI:** Ubuntu Server 22.04 LTS
- **Type:** t3.small or higher
- **Security Group:** Allow SSH (22) from Controller's private IP

### c. Jenkins Agent 2 (Oracle backend)
- **AMI:** Ubuntu Server 22.04 LTS
- **Type:** t3.small or higher (adjust for Oracle workload)
- **Security Group:** Allow SSH (22) from Controller's private IP

## 2. Prepare All Servers

### Update and Install Java
On all three servers:

```sh
sudo apt update
sudo apt upgrade -y
sudo apt install openjdk-17-jdk -y
java -version
```

### Set Hostnames (optional, for clarity)

```sh
sudo hostnamectl set-hostname jenkins-controller # On controller
sudo hostnamectl set-hostname jenkins-agent-java # Agent 1
sudo hostnamectl set-hostname jenkins-agent-oracle # Agent 2
```

## 3. Install Jenkins on Controller

```sh
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb https://pkg.jenkins.io/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

- **Open port 8080** in AWS Security Group to your IP
- Access Jenkins at `http://<controller-public-ip>:8080`
- Get the initial admin password:


```sh
  sudo cat /var/lib/jenkins/secrets/initialAdminPassword
  ```
- Complete the web setup wizard, install suggested plugins, create first admin user

## 4. Configure SSH for Agent Connection

### a. Create Jenkins User on Agents
On both agents:
```

sh
sudo adduser jenkins
sudo usermod -aG sudo jenkins


```

### b. Generate SSH Key on Controller
On the controller:
```

sh
sudo -i -u jenkins
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_jenkins_agents
cat ~/.ssh/id_rsa_jenkins_agents.pub

```

### c. Copy Public Key to Agents
On each agent, as the `jenkins` user:
```

su - jenkins
mkdir -p ~/.ssh
echo "<controller-public-key>" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh

```
(Replace `<controller-public-key>` with the content from the previous step.)

- **Test SSH from controller to each agent as jenkins:**
  ```sh
  ssh -i ~/.ssh/id_rsa_jenkins_agents jenkins@<agent-ip>
  ```

## 5. Add Agent Nodes in Jenkins UI
1. Go to **Manage Jenkins > Manage Nodes and Clouds > New Node**
2. Name: `java-agent`, Type: Permanent Agent
3. Set # of executors, Remote root directory: `/home/jenkins`
4. Labels: `java`
5. Launch method: **Launch agents via SSH**
   - Host: `<agent1-private-ip>`
   - Credentials: Add new → SSH Username with private key
     - Username: `jenkins`
     - Private Key: Use the contents of `~jenkins/.ssh/id_rsa_jenkins_agents` from the controller
6. Repeat for `oracle-agent`, label it as `oracle`

## 6. Install Build Tools on Agents

### On Java Agent:
```

sh
sudo apt install git maven gradle -y

```

### On Oracle Agent:
- Install any necessary Oracle client/SDK/tools for backend builds
- Example for SQL*Plus or Oracle Instant Client:
  ```sh
  sudo apt install alien libaio1
  wget https://download.oracle.com/otn_software/linux/instantclient/instantclient-basic-linux.x64-21.6.0.0.0dbru.zip
  # Unzip and configure as needed
  ```

## 7. Create Jenkins Pipeline Jobs

### Example: Java Build Job (runs on `java-agent`)
```

groovy
pipeline {
    agent { label 'java' }
    stages {
        stage('Checkout') { steps { git 'https://github.com/your-org/java-repo.git' } }
        stage('Build') { steps { sh 'mvn clean package' } }
        stage('Archive') { steps { archiveArtifacts 'target/*.jar' } }
    }
}

```

### Example: Oracle Backend Job (runs on `oracle-agent`)
```

groovy
pipeline {
    agent { label 'oracle' }
    stages {
        stage('Checkout') { steps { git 'https://github.com/your-org/oracle-backend.git' } }
        stage('Build') { steps { sh './build-oracle.sh' } }
        stage('Archive') { steps { archiveArtifacts 'output/*.zip' } }
    }
}

```

## 8. Connect Jenkins to GitHub
- Install **GitHub** and **Pipeline** plugins
- Add a GitHub Personal Access Token as a Jenkins credential
- In each Jenkins job, configure GitHub repo URL and set up webhook:
  - GitHub > Settings > Webhooks > Payload URL: `http://<controller-public-ip>:8080/github-webhook/`
  - Content type: `application/json`
  - Events: Push (or as needed)

## 9. Test Your Setup
- Push code to GitHub
- Verify Jenkins triggers job via webhook
- Confirm the job runs on the correct agent (Java or Oracle) by label
- Artifacts/results are archived and visible in Jenkins UI

## 10. (Optional) Secure and Harden Jenkins
- Set up HTTPS (using reverse proxy like Nginx or Apache)
- Restrict security groups to trusted networks
- Set up LDAP/SSO for authentication
- Regularly update and backup Jenkins

## Summary Diagram
```

GitHub --(webhook)--> Jenkins Controller (Ubuntu EC2)
           |                    |
           |                    +--- SSH ---> Java Agent (Ubuntu EC2)
           |                    |
           |                    +--- SSH ---> Oracle Agent (Ubuntu EC2)
```
