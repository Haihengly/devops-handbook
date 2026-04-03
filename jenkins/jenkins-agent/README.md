# Jenkins Agent Setup Tutorial

> This tutorial explains how to **create and run a permanent Jenkins agent** on a Linux VM using systemd.

---

## What is RBAC?

> It’s a computer (VM, server, or container) that connects to the Jenkins server.

>The agent actually runs the jobs — it does the heavy work, so the main Jenkins server stays clean and responsive.

---

## ⚙️ Requirements

- Jenkins server up and running
- Linux VM for agent
- SSH access to the VM
- Java installed on the VM (`openjdk-17-jdk` recommended)

---

## Step 1: Install Java

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
java -version
```

---

## Step 2: Create a Jenkins user

```bash
sudo useradd -m -s /bin/bash jenkins
sudo passwd jenkins
```

---

## Step 3: Download agent.jar

```bash
mkdir -p /home/jenkins
cd /home/jenkins
curl -sO http://<YOUR_JENKINS_URL>/jnlpJars/agent.jar
```

---

## Step 4: Get agent secret from Jenkins

1. Jenkins → Manage Jenkins → Manage Nodes → New Node  
2. Create a permanent agent  
3. Copy the secret key for JNLP agent

---

## Step 5: Test run manually

```bash
java -jar agent.jar -url http://<YOUR_JENKINS_URL>/ -secret <SECRET> -name Agent01 -webSocket -workDir "/home/jenkins"
```

---

## Step 6: Create systemd service

```bash
sudo nano /etc/systemd/system/jenkins-agent.service
```

Paste:

```ini
[Unit]
Description=Jenkins Agent
After=network.target

[Service]
User=jenkins
WorkingDirectory=/home/jenkins
ExecStart=/usr/bin/java -jar /home/jenkins/agent.jar \
  -url http://<YOUR_JENKINS_URL>/ \
  -secret <SECRET> \
  -name Agent01 \
  -webSocket \
  -workDir /home/jenkins
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## Step 7: Enable and start service

```bash
sudo systemctl daemon-reload
sudo systemctl enable jenkins-agent
sudo systemctl start jenkins-agent
sudo systemctl status jenkins-agent
```

---

## Step 8: Update agent.jar

```bash
cd /home/jenkins
curl -sO http://<YOUR_JENKINS_URL>/jnlpJars/agent.jar
sudo systemctl restart jenkins-agent
sudo systemctl status jenkins-agent
```

---