# 🚀 Jenkins CI/CD Pipeline with Maven, Amazon S3 & Tomcat

A practical CI/CD project demonstrating how to automate the build, artifact storage, and deployment of a Java/Maven web application using **GitHub, Jenkins, Maven, Amazon S3, and Apache Tomcat**.

> **Learning Project:** The Java application source used in this project was forked from a trainer's repository for learning purposes. The CI/CD infrastructure, Jenkins configuration, AWS integration, artifact management, and Tomcat deployment were implemented and tested as part of this project.

---

## 🏗️ Architecture

```text
                    ┌──────────────┐
                    │    GitHub    │
                    │  Source Code │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │   Jenkins    │
                    │     CI/CD    │
                    └──────┬───────┘
                           │
                     Maven Build
                           │
                           ▼
                    ┌──────────────┐
                    │   WAR File   │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │  Amazon S3   │
                    │   Artifact   │
                    │   Storage    │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │    Tomcat    │
                    │  EC2 Server  │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Java Web App │
                    └──────────────┘
```

---

## 🔄 CI/CD Workflow

```text
Developer
    │
    ▼
GitHub
    │
    ▼
Jenkins
    │
    ├── Checkout Source Code
    │
    ├── Maven Compile
    │
    ├── Maven Test
    │
    ├── Maven Package
    │
    └── Generate WAR
            │
            ▼
       Amazon S3
      WAR Artifact
            │
            ▼
       Tomcat EC2
            │
            ▼
       Deploy WAR
```

---

## 🛠️ Technologies Used

| Technology                | Purpose                         |
| ------------------------- | ------------------------------- |
| **GitHub**                | Source code management          |
| **Jenkins**               | CI/CD automation                |
| **Apache Maven**          | Build and dependency management |
| **Java 17**               | Application runtime             |
| **Java 21**               | Jenkins runtime                 |
| **Amazon EC2**            | Jenkins and Tomcat servers      |
| **Amazon S3**             | WAR artifact storage            |
| **Apache Tomcat 10.1.59** | Application server              |
| **AWS IAM**               | Secure AWS permissions          |
| **Linux**                 | Server administration           |

---

## 📋 Prerequisites

Before starting this project, make sure you have:

* AWS account
* GitHub account
* Jenkins server
* Tomcat server
* Java
* Maven
* Git
* An AWS S3 bucket
* Appropriate AWS IAM roles

---

# ☁️ AWS Infrastructure

The project uses two EC2 instances.

### Jenkins Server

The Jenkins EC2 instance contains:

```text
Java 21
Maven
Jenkins
Git
AWS CLI
```

### Tomcat Server

The Tomcat EC2 instance contains:

```text
Java 17
Apache Tomcat 10.1.59
```

---

# 🔐 IAM Configuration

IAM roles are used instead of storing AWS access keys on the EC2 instances.

### Jenkins EC2 Role

The Jenkins server requires permissions to upload and manage artifacts in S3.

Example permissions:

```text
s3:ListBucket
s3:GetObject
s3:PutObject
s3:DeleteObject
```

### Tomcat EC2 Role

The Tomcat server requires permission to retrieve deployment artifacts from S3.

Example:

```text
s3:GetObject
```

This approach avoids storing AWS access keys directly inside the Jenkins project.

---

# ⚙️ Jenkins Configuration

The Jenkins project was initially implemented using a **Freestyle Project**.

The main stages are:

```text
1. Source Code Checkout
2. Maven Build
3. Test
4. WAR Generation
5. Upload Artifact to S3
6. Deploy Application to Tomcat
```

### Maven Build

The project is built using:

```bash
mvn clean package
```

The generated WAR file is created inside:

```text
target/
```

Example:

```text
target/hoststar.war
```

---

# 📦 Amazon S3 Artifact Storage

After Maven generates the WAR file, the artifact can be uploaded to an S3 bucket.

Example:

```bash
aws s3 cp target/*.war s3://YOUR-BUCKET/
```

S3 is used as centralized artifact storage.

This makes it possible to keep the build artifact separately from the Jenkins workspace.

---

# 🐱 Tomcat Configuration

Apache Tomcat is installed on a separate EC2 instance.

Tomcat location:

```text
/root/apache-tomcat-10.1.59
```

The Tomcat Manager application is configured with the `manager-script` role so Jenkins can perform automated deployments.

Example configuration:

```xml
<role rolename="manager-script"/>

<user username="jenkins"
      password="CHANGE_THIS_PASSWORD"
      roles="manager-script"/>
```

Jenkins stores these credentials using the Jenkins Credentials Manager rather than hard-coding them into the project.

---

# 🔑 Jenkins Credentials

Create a Jenkins credential:

```text
Manage Jenkins
    ↓
Credentials
    ↓
Global
    ↓
Add Credentials
```

Configuration:

```text
Kind: Username with password

Username: jenkins

Password: <Tomcat Manager password>

ID: tomcat-manager
```

---

# 🔌 Jenkins Plugins

The project can use the following Jenkins plugins:

* **Git Plugin** — GitHub source checkout
* **Maven Integration Plugin** — Maven build integration
* **Deploy to Container Plugin** — WAR deployment to Tomcat
* **Pipeline Plugin** — required when converting the project to Jenkins Pipeline
* **SSH Agent Plugin** — required only when deployment uses SSH authentication

> SSH Agent is not required when Jenkins deploys through the Tomcat Manager API using username/password authentication.

---

# 🌐 Jenkins → Tomcat Communication

The deployment communication follows:

```text
Jenkins EC2
     │
     │ HTTP :8080
     ▼
Tomcat EC2
     │
     ▼
Tomcat Manager
```

The Tomcat Manager application normally restricts remote access.

For this lab, the Manager configuration was adjusted to allow Jenkins to communicate with Tomcat.

For production environments, access should be restricted using AWS Security Groups and Tomcat access controls instead of exposing the Manager application publicly.

---

# 📁 Project Structure

```text
java-project-maven-new/
│
├── src/
│   ├── main/
│   └── test/
│
├── pom.xml
├── README.md
└── INSTALL.md
```

---

# 🚀 Installation

Detailed installation instructions for Jenkins and Tomcat are available in:

```text
INSTALL.md
```

The installation guide covers:

* Jenkins installation
* Java 21
* Maven installation
* Tomcat installation
* Java 17
* Tomcat Manager configuration
* Jenkins credentials
* Remote Tomcat access
* Troubleshooting

---

# 🧪 Build Locally

Clone the repository:

```bash
git clone https://github.com/nexqor/java-project-maven-new.git
```

Enter the project directory:

```bash
cd java-project-maven-new
```

Build the application:

```bash
mvn clean package
```

Run tests:

```bash
mvn test
```

The WAR file will be generated inside:

```text
target/
```

---

# 🔄 Future Improvements

The current implementation uses a Jenkins Freestyle Project.

Possible improvements include:

* Convert Freestyle project to Jenkins Pipeline
* Add a `Jenkinsfile`
* Configure GitHub Webhooks
* Add automated testing
* Add Docker-based deployment
* Add Terraform infrastructure
* Add SonarQube code analysis
* Add Trivy security scanning
* Add Docker image scanning
* Implement blue-green deployment
* Add monitoring and logging
* Implement AWS deployment automation

---

# 🔒 Security Considerations

Do **not** commit the following files or secrets:

```text
*.pem
.env
AWS access keys
AWS secret keys
Tomcat passwords
Jenkins passwords
Private SSH keys
```

Use:

* AWS IAM Roles
* Jenkins Credentials
* AWS Security Groups
* Strong authentication
* Least-privilege IAM policies

---

# 🎯 What I Learned

Through this project, I practiced:

* Jenkins installation and configuration
* Jenkins Freestyle Projects
* GitHub integration
* Maven build automation
* Java WAR packaging
* Amazon S3 artifact storage
* AWS IAM roles
* EC2 administration
* Apache Tomcat configuration
* Tomcat Manager authentication
* Automated application deployment
* CI/CD fundamentals
* Linux server administration

---

## 👨‍💻 Author

**Manish Kumar**

B.Tech — Cloud Technology & Information Security

Focused on:

```text
AWS DevOps
Cloud Security
DevSecOps
CI/CD
Linux
Infrastructure as Code
Cloud Automation
```

GitHub:

```text
https://github.com/nexqor
```

---

## 📌 Disclaimer

This repository is a learning project.

The Java application source was forked from a trainer's repository for educational purposes. The DevOps implementation—including Jenkins configuration, Maven automation, AWS S3 artifact storage, IAM configuration, and Tomcat deployment—was performed as part of this learning project.
