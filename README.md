<div align="center">

# DevSecOps Banking Application

A high-performance, containerized financial platform built with Spring Boot 3, Java 21, and integrated Contextual AI. This project implements a secure "DevSecOps Pipeline" using GitHub Actions, OIDC authentication, and AWS managed services.

[![Java Version](https://img.shields.io/badge/Java-21-blue.svg)](https://www.oracle.com/java/technologies/javase/jdk21-archive-downloads.html)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.4.1-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-orange.svg)](.github/workflows/devsecops.yml)
[![AWS OIDC](https://img.shields.io/badge/Security-OIDC-red.svg)](#phase-3-security-and-identity-configuration)

</div>

<img width="841" height="442" alt="image" src="https://github.com/user-attachments/assets/cae77a97-893a-4236-9df7-01296ac619dd" />


---

## Technical Architecture

The application is deployed across a multi-tier, segmented AWS environment. The control plane leverages GitHub Actions with integrated security gates at every stage.

```mermaid
graph TD
    subgraph "External Control Plane"
        GH[GitHub Actions]
        User[User Browser]
    end

    subgraph "AWS Infrastructure (VPC)"
        subgraph "Application Tier"
            AppEC2[App EC2 - Ubuntu/Docker]
            DB[(MySQL 8.0 Container)]
        end

        subgraph "Artificial Intelligence Tier"
            Ollama[Ollama EC2 - AI Engine]
        end

        subgraph "Identity & Secrets"
            Secrets[AWS Secrets Manager]
            OIDC[IAM OIDC Provider]
        end

        subgraph "Registry"
            ECR[Amazon ECR]
        end
    end

    GH -->|1. OIDC Authentication| OIDC
    GH -->|2. Push Scanned Image| ECR
    GH -->|3. SSH Orchestration| AppEC2
    GH -->|4. DAST Scan| AppEC2
    
    User -->|Port 8080| AppEC2
    AppEC2 -->|JDBC Connection| DB
    AppEC2 -->|REST Integration| Ollama
    AppEC2 -->|Runtime Secrets| Secrets
    AppEC2 -->|Pull Image| ECR
```

---

## Security Pipeline (DevSecOps Pipeline)

The CI/CD pipeline enforces **9 sequential security gates** before any code reaches production:

| Gate | Name | Tool | Purpose |
| :---: | :--- | :--- | :--- |
| 1 | Secret Scan | Gitleaks | Scans entire Git history for leaked secrets |
| 2 | Lint | Checkstyle | Enforces Java Google-Style coding standards |
| 3 | SAST | Semgrep | Scans Java source code for security flaws and OWASP Top 10 |
| 4 | SCA | OWASP Dependency Check (first time run can take more than 30+ minutes) | Scans Maven dependencies for known CVEs |
| 5 | Build | Maven | Compiles and packages the application |
| 6 | Container Scan | Trivy | Scans the Docker image for OS and library vulnerabilities |
| 7 | Push | Amazon ECR | Pushes the image only after Trivy passes |
| 8 | Deploy | SSH / Docker Compose | Automated deployment to AWS EC2 |
| 9 | DAST | OWASP ZAP | Dynamic attack surface scanning on live app |

---

## Technology Stack

- **Backend Framework**: Java 21, Spring Boot 3.4.1
- **Security Strategy**: Spring Security, IAM OIDC, Secrets Manager
- **Persistence Layer**: MySQL 8.0 (Docker Container)
- **AI Integration**: Ollama (TinyLlama)
- **DevOps Tooling**: Docker, Docker Compose, GitHub Actions, AWS CLI, jq
- **Infrastructure**: Amazon EC2, Amazon ECR, Amazon VPC

---

## Implementation Phases

### Phase 1: AWS Infrastructure Initialization

1. **Container Registry (ECR)**:

   - Establish a private ECR repository named `devsecops-bankapp`.

      <img width="762" height="120" alt="image" src="https://github.com/user-attachments/assets/23469aa4-b2f8-455b-a90d-c66fc6130c4a" />


2. **Application Server (EC2)**:

   - Deploy an Ubuntu 22.04 instance with below `User Data`.

      ```bash
      #!/bin/bash

      sudo apt update 
      sudo apt install -y docker.io docker-compose-v2 jq
      sudo usermod -aG docker ubuntu
      sudo newgrp docker
      sudo snap install aws-cli --classic
      ```

   - Configure Security Group to open inbound rule for Port 22 (Management) and Port 8080 (Service).

      > Better to give `name` to Security Group created.

   - Create an IAM Instance Profile(IAM EC2 role) containing permissions:
     - `AmazonEC2ContainerRegistryPowerUser`
     - `AWSSecretsManagerClientReadOnlyAccess`

        <img width="735" height="175" alt="image" src="https://github.com/user-attachments/assets/59c652cf-8c71-41cd-a84f-f77f9675a7a4" />


   - Attach it to Application EC2. Select EC2 -> Actions -> Security -> Modify IAM role -> Attach created IAM role.

      <img width="760" height="140" alt="image" src="https://github.com/user-attachments/assets/100e7cb1-d073-4d16-8933-e0264287c011" />

   
   - Connect to EC2 Instance and Run below command to check whether IAM role is working or not.

      ```bash
      aws sts get-caller-identity
      ```

      You will get your account details with IAM role assumed.

3. **AI Engine Tier (Ollama)**:
   - Deploy a dedicated Ubuntu EC2 instance.
   - Open Inbound Port `11434` from the Application EC2 Security Group.

      > Better to give `name` to Security Group created.
        
      <img width="777" height="280" alt="image" src="https://github.com/user-attachments/assets/81618a08-10d6-444e-9771-9ee61fbe9474" />


   - Automate initialization using the [ollama-setup.sh](scripts/ollama-setup.sh) script via EC2 User Data.
    
     <img width="777" height="490" alt="image" src="https://github.com/user-attachments/assets/3ea6180c-7d36-468a-b5b1-818d7c4eb7e7" />


   - Verify the AI engine is responsive and the model is pulled in `AI engine EC2`:

     ```bash
     ollama list
     ```

      <img width="576" height="86" alt="image" src="https://github.com/user-attachments/assets/3975e1de-c13f-4d26-834f-1b3740215232" />


---

### Phase 2: Security and Identity Configuration

The deployment pipeline utilizes OpenID Connect (OIDC) for secure, keyless authentication between GitHub and AWS.

1. **IAM Identity Provider**:
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`

      <img width="777" height="352" alt="image" src="https://github.com/user-attachments/assets/c117a610-0fae-4f6a-a5b0-93dd98d5c760" />


2. **Deployment Role**:
   - Click on created `Identity provider`
   - Asign & Create a role named `GitHubActionsRole`.
   - Enter following details:
      - `Identity provider`: Select created one.
      - `Audience`: Select created one.
      - `GitHub organization`: Your GitHub Username or Orgs Name where this repo is located.
      - `GitHub repository`: Write the Repository name of this project. `(e.g, DevSecOps-Bankapp)`
      - `GitHub branch`: branch to use for this project `(e.g, devsecops)`
      - Click on `Next`

      <img width="782" height="455" alt="image" src="https://github.com/user-attachments/assets/e0dbf361-2bc3-43f3-a165-79d329223efc" />


   - Assign `AmazonEC2ContainerRegistryPowerUser` permissions.

      <img width="777" height="236" alt="image" src="https://github.com/user-attachments/assets/4bf99cfa-d3ca-45c0-8b1d-09765eb62cdd" />


   - Click on `Next`, Enter name of role and click on `Create role`.

      <img width="780" height="502" alt="image" src="https://github.com/user-attachments/assets/4a17141c-4653-4cef-9df4-f03263138c3e" />


---

### Phase 3: Secrets and Pipeline Configuration

#### 1. AWS Secrets Manager
Create a secret named `bankapp/prod-secrets` in `Other type of secret` with the following key-value pairs:

| Secret Key | Description | Sample/Default Value |
| :--- | :--- | :--- |
| `DB_HOST` | The MySQL container service name | `db` |
| `DB_PORT` | The database port | `3306` |
| `DB_NAME` | The application database name | `bankappdb` |
| `DB_USER` | The database username | `bankuser` |
| `DB_PASSWORD` | The database password | `Test@123` |
| `OLLAMA_URL` | The private URL for the AI tier | `http://<PRIVATE-IP>:11434` |

<img width="827" height="401" alt="image" src="https://github.com/user-attachments/assets/2ab93bfa-a5ef-44c8-85e2-fd4d377a84e6" />


#### 2. GitHub Repository Secrets
Configure the following Action Secrets within your GitHub repository settings:

| Secret Name | Description |
| :--- | :--- |
| `AWS_ROLE_ARN` | The ARN of the `GitHubActionsRole` |
| `AWS_REGION` | The AWS region where resources are deployed |
| `AWS_ACCOUNT_ID` | Your 12-digit AWS account number |
| `ECR_REPOSITORY` | The name of the ECR repository (`devsecops-bankapp`) |
| `EC2_HOST` | The public IP address of the Application EC2 |
| `EC2_USER` | The SSH username (default is `ubuntu`) |
| `EC2_SSH_KEY` | The content of your private SSH key (`.pem` file) |
| `NVD_API_KEY` | Free API key from [nvd.nist.gov](https://nvd.nist.gov/developers/request-an-api-key) for OWASP SCA scans |

> **Note**: The `NVD_API_KEY` raises the NVD API rate limit from ~5 requests/30s to 50 requests/30s, reducing the OWASP Dependency Check scan time from 30+ minutes to under 8 minutes. Without it the SCA job will time out.

---

## Continuous Integration and Deployment

The DevSecOps lifecycle is orchestrated through the [DevSecOps Main Pipeline](.github/workflows/devsecops-main.yml), which securely sequences three modular workflows: [CI](.github/workflows/ci.yml), [Build](.github/workflows/build.yml), and [CD](.github/workflows/cd.yml). Together they enforce **9 sequential security gates** before any code reaches production. Every `git push` to the `main` or `devsecops` branch triggers the full pipeline automatically.

| Gate | Job | Tool | Action |
| :---: | :--- | :--- | :--- |
| 1 | `gitleaks` | Gitleaks | **Strict**: Fails if any secrets are found in history. |
| 2 | `lint` | Checkstyle | **Audit**: Reports style violations but doesn't block (Google Style). |
| 3 | `sast` | Semgrep | **Strict**: Scans code for vulnerabilities. Fails on findings. |
| 4 | `sca` | OWASP Dependency Check | **Strict**: Fails if any dependency has CVSS > 7.0. |
| 5 | `build` | Maven | Standard build and test stage. |
| 6 | `image_scan` | Trivy | **Strict**: Scans Docker image layers. Fails on any High/Critical CVE. |
| 7 | `push_to_ecr` | Amazon ECR | Pushes the verified image to AWS ECR using OIDC. |
| 8 | `deploy` | SSH / Docker Compose | Fetches secrets from AWS Secrets Manager and recreates the container. |
| 9 | `dast` | OWASP ZAP | **Audit Mode**: Comprehensive scan that reports findings as artifacts, but does not block the pipeline. |

All scan reports (OWASP, Trivy, ZAP) are uploaded as downloadable **Artifacts** in each GitHub Actions run, YOu can look into the **Artifacts**.

- CI/CD

   <img width="797" height="142" alt="image" src="https://github.com/user-attachments/assets/34cae089-d24c-4c47-8707-9d60ad8ae813" />


- Artifacts

  <img width="805" height="435" alt="image" src="https://github.com/user-attachments/assets/ffe9d40c-1180-40e4-99e2-db20ffd50c18" />

   
---

## Operational Verification


- **Application Working**:

  <img width="812" height="452" alt="image" src="https://github.com/user-attachments/assets/4c767dfc-7277-42fa-9f1f-9b184f5c1a71" />


- **Database Connectivity**: 

  ```bash
  docker exec -it db mysql -u <USER> -p bankappdb -e "SELECT * FROM accounts;"
  ```

  <img width="815" height="132" alt="image" src="https://github.com/user-attachments/assets/ea2abca5-e071-4e7c-8a64-16928a041302" />


  > **ZAP** is automatically created by **DAST - OWASP ZAP Baseline Scan** job in [cd.yml](.github/workflows/cd.yml). Read more about it(How, Why it does) on google...

- **Network Validation**: 

  ```bash
  nc -zv <OLLAMA-PRIVATE-IP> 11434
  ```

  <img width="562" height="95" alt="image" src="https://github.com/user-attachments/assets/11a91a30-e353-45f9-ba0b-c3292a93bcea" />


---

<div align="center">



</div>
