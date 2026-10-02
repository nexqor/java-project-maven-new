# Jenkins + Maven + Tomcat Installation Guide

This guide explains how to set up a Jenkins server and a Tomcat server for a Java/Maven CI/CD deployment environment.

## Architecture

```text
GitHub
   │
   ▼
Jenkins EC2
   │
   ├── Git
   ├── Maven
   └── Java 21
   │
   ▼
Build WAR
   │
   ▼
Tomcat EC2
   │
   └── Apache Tomcat 10.1
```

---

# 1. Jenkins Server Setup

## Prerequisites

* Amazon Linux EC2 instance
* Root or sudo access
* Java 21
* Maven
* Internet connectivity
* Jenkins port `8080` allowed in the EC2 Security Group

## Jenkins Installation Script

Create the installation script:

```bash
vi jenkins.sh
```

Add:

```bash
#!/bin/bash

echo "************** Jenkins Installation **************"

sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/rpm/jenkins.repo

sudo yum upgrade -y

# Install Java 21 and required dependencies
sudo yum install fontconfig java-21-amazon-corretto -y

# Install Jenkins
sudo yum install jenkins -y

echo "************** Maven Installation **************"

sudo yum install maven -y

echo "************** Starting Jenkins **************"

sudo systemctl enable jenkins
sudo systemctl start jenkins

sudo systemctl status jenkins
```

Make the script executable:

```bash
chmod +x jenkins.sh
```

Run it:

```bash
./jenkins.sh
```

## Verify Java

```bash
java -version
```

Expected Java version:

```text
Java 21
```

## Verify Maven

```bash
mvn -version
```

## Verify Jenkins

```bash
sudo systemctl status jenkins
```

Jenkins should show:

```text
active (running)
```

Jenkins can then be accessed through:

```text
http://<JENKINS-PUBLIC-IP>:8080
```

---

# 2. Tomcat Server Setup

## Prerequisites

* Amazon Linux EC2 instance
* Java 17
* Internet connectivity
* Port `8080` configured in the EC2 Security Group

## Install Java 17

```bash
sudo dnf install java-17-amazon-corretto-devel -y
```

Verify:

```bash
java -version
```

---

## Download Apache Tomcat

Download Tomcat 10.1.59:

```bash
wget https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.59/bin/apache-tomcat-10.1.59.tar.gz
```

Extract the archive:

```bash
tar -xzf apache-tomcat-10.1.59.tar.gz
```

Tomcat will be available at:

```text
/root/apache-tomcat-10.1.59
```

---

# 3. Configure Tomcat Manager

Jenkins needs to communicate with Tomcat's Manager application to deploy the WAR file.

Navigate to:

```bash
cd /root/apache-tomcat-10.1.59/conf
```

Edit:

```bash
vi tomcat-users.xml
```

Add the following configuration inside the `<tomcat-users>` element:

```xml
<tomcat-users>
    <role rolename="manager-gui"/>
    <role rolename="manager-script"/>
    <user username="jenkins"
          password="CHANGE_THIS_PASSWORD"
          roles="manager-script"/>
</tomcat-users>
```

### Roles

* `manager-gui` — allows access to the Tomcat Manager web interface.
* `manager-script` — allows scripted deployment through the Manager API.

For Jenkins automated deployment, `manager-script` is the important role.

> Replace `CHANGE_THIS_PASSWORD` with a strong password.

Do **not** commit `tomcat-users.xml` or the password to GitHub.

---

# 4. Allow Remote Access to Tomcat Manager

By default, Tomcat Manager restricts access to localhost.

Navigate to:

```bash
cd /root/apache-tomcat-10.1.59/webapps/manager/META-INF
```

Open:

```bash
vi context.xml
```

The file contains a `RemoteAddrValve` similar to:

```xml
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
       allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
```

This restriction allows Manager access only from localhost.

For a lab environment where Jenkins is running on a different EC2 instance, this prevents Jenkins from accessing the Manager application.

For testing, remove or comment out this restriction.

Example:

```xml
<!--
<Valve className="org.apache.catalina.valves.RemoteAddrValve"
       allow="127\.\d+\.\d+\.\d+|::1|0:0:0:0:0:0:0:1" />
-->
```

### Why is this required?

The Jenkins server and Tomcat server are separate machines.

```text
Jenkins EC2
     │
     │ HTTP
     ▼
Tomcat EC2
     │
     ▼
Tomcat Manager
```

The default `RemoteAddrValve` allows only local connections. Therefore, Jenkins' request can be rejected even when the username and password are correct.

> **Security recommendation:** For a real production environment, do not simply allow unrestricted public access to Tomcat Manager. Prefer restricting port `8080` in the EC2 Security Group to the Jenkins server's private IP/security group, and configure the Tomcat access rule to allow only the Jenkins host.

---

# 5. Start Tomcat

Navigate to the Tomcat `bin` directory:

```bash
cd /root/apache-tomcat-10.1.59/bin
```

Start Tomcat:

```bash
./startup.sh
```

You should see output similar to:

```text
Tomcat started.
```

To stop Tomcat:

```bash
./shutdown.sh
```

---

# 6. Verify Tomcat

Open:

```text
http://<TOMCAT-PUBLIC-IP>:8080
```

The Apache Tomcat welcome page should appear.

---

# 7. Verify Tomcat Manager

Open:

```text
http://<TOMCAT-PUBLIC-IP>:8080/manager/
```

Use the credentials configured in `tomcat-users.xml`.

```text
Username: jenkins
Password: <your-password>
```

For Jenkins automated deployment, the `manager-script` role is used.

---

# 8. Jenkins Credentials

In Jenkins:

```text
Manage Jenkins
    ↓
Credentials
    ↓
Global
    ↓
Add Credentials
```

Select:

```text
Kind: Username with password
```

Use:

```text
Username: jenkins
Password: <same password configured in Tomcat>
```

Recommended credential ID:

```text
tomcat-manager
```

Do not hard-code the Tomcat username or password inside a Jenkinsfile.

---

# 9. Jenkins Maven Build

A typical Maven project contains:

```text
pom.xml
src/
```

Jenkins can build the project using:

```bash
mvn clean package
```

The generated WAR file will normally be located in:

```text
target/
```

Example:

```text
target/hoststar.war
```

---

# 10. Deployment Flow

The complete deployment workflow is:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
Jenkins
    │
    ├── Checkout
    │
    ├── Maven Build
    │
    ├── Run Tests
    │
    └── Generate WAR
    │
    ▼
Amazon S3
    │
    │ Artifact Storage
    ▼
Tomcat EC2
    │
    ▼
Tomcat Manager
    │
    ▼
Java Web Application
```

---

# 11. Troubleshooting

## Jenkins is not running

Check:

```bash
sudo systemctl status jenkins
```

Restart:

```bash
sudo systemctl restart jenkins
```

View logs:

```bash
sudo journalctl -u jenkins -f
```

---

## Maven command not found

Check:

```bash
mvn -version
```

If Maven is not installed:

```bash
sudo yum install maven -y
```

---

## Tomcat is not accessible

Check whether Tomcat is running:

```bash
ps -ef | grep tomcat
```

Check port:

```bash
ss -lntp | grep 8080
```

Also verify that the AWS Security Group allows TCP port `8080`.

---

## Tomcat Manager returns 401 Unauthorized

Check:

```text
/root/apache-tomcat-10.1.59/conf/tomcat-users.xml
```

Verify:

```xml
<role rolename="manager-script"/>
<user username="jenkins"
      password="YOUR_PASSWORD"
      roles="manager-script"/>
```

Then restart Tomcat:

```bash
cd /root/apache-tomcat-10.1.59/bin
./shutdown.sh
./startup.sh
```

---

## Tomcat Manager returns 403 Forbidden

Check:

```text
/root/apache-tomcat-10.1.59/webapps/manager/META-INF/context.xml
```

The `RemoteAddrValve` may be blocking the Jenkins server.

For a lab setup, comment out the restriction or configure it to allow the Jenkins server's IP.

Also check the AWS Security Group.

---

## Jenkins cannot connect to Tomcat

Verify:

```text
Jenkins EC2
     ↓
TCP 8080
     ↓
Tomcat EC2
```

Check:

1. Tomcat is running.
2. Port `8080` is listening.
3. AWS Security Group allows Jenkins → Tomcat traffic.
4. Tomcat Manager is accessible.
5. Jenkins credentials match `tomcat-users.xml`.
6. `manager-script` role is configured.

---

# 12. Security Notes

For learning/lab environments:

* Use IAM roles instead of AWS access keys on EC2.
* Never commit passwords, private keys, or AWS credentials.
* Never commit `.pem` files.
* Do not expose Tomcat Manager to the entire internet in a production environment.
* Restrict port `8080` to trusted sources.
* Use strong passwords for Tomcat Manager.
* Store deployment credentials in Jenkins Credentials.
* Consider HTTPS and a reverse proxy for production deployments.

---

# 13. Useful Commands

### Jenkins

```bash
sudo systemctl start jenkins
sudo systemctl stop jenkins
sudo systemctl restart jenkins
sudo systemctl status jenkins
```

### Maven

```bash
mvn clean package
mvn test
mvn -version
```

### Tomcat

```bash
cd /root/apache-tomcat-10.1.59/bin

./startup.sh
./shutdown.sh
```

### Check Tomcat process

```bash
ps -ef | grep tomcat
```

### Check port 8080

```bash
ss -lntp | grep 8080
```

---

## Final Environment

| Component             | Configuration         |
| --------------------- | --------------------- |
| Jenkins               | Jenkins on EC2        |
| Java                  | Amazon Corretto 21    |
| Maven                 | Apache Maven          |
| Tomcat                | Apache Tomcat 10.1.59 |
| Tomcat Java           | Amazon Corretto 17    |
| Application           | Java/Maven WAR        |
| Artifact Storage      | Amazon S3             |
| Deployment            | Jenkins → Tomcat      |
| Tomcat Authentication | `manager-script`      |
| Jenkins Credential    | Username + Password   |
