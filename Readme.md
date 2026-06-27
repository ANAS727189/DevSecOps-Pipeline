# End-to-End DevSecOps CI/CD Pipeline

This project demonstrates a comprehensive DevSecOps lifecycle, integrating automated infrastructure provisioning, security scanning, containerized deployment, and real-time observability.

---

## 1. Pipeline Architecture Flow

The workflow follows a strict sequence to ensure that code is secured, built, and deployed to a scalable environment:

1.  **Source Code Management:** Developer pushes code to GitHub.
2.  **Continuous Integration (Jenkins):** Triggered by the push, Jenkins initiates the pipeline.
3.  **Static Security Analysis:** Trivy scans the source code filesystem for vulnerable dependencies.
4.  **Artifact Containerization:** Docker builds the application image and pushes it to DockerHub.
5.  **Dynamic Security Analysis:** Trivy scans the runtime Docker image for OS-level vulnerabilities.
6.  **Infrastructure as Code (Terraform):** Provisions an AWS EC2 instance with a 20GB EBS volume and specific Security Group rules.
7.  **Dynamic Provisioning (Ansible):** Connects to the new EC2 instance via SSH to install Docker and deploy the application and monitoring stack.
8.  **Observability:** Prometheus scrapes hardware and container metrics, visualized through Grafana dashboards.



---

## 2. Infrastructure Setup

### AWS Configuration
A dedicated IAM user must be created with `AmazonEC2FullAccess` and `AmazonVPCFullAccess`. An RSA Key Pair named `devops-key` is required for SSH access.

### Jenkins Environment (Docker-based)
Jenkins is executed as a container with the host's Docker socket mounted to allow Docker commands from within the container:

```bash
docker run -d -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --name jenkins jenkins/jenkins:lts
```

### Setup in Windows/Linux

1.  Install Docker Desktop (Windows) or Docker Engine (Linux).
2. Pull and run the Jenkins container as shown above.
  
```bash
  # In PowerShell (run as Administrator)
docker run -d -p 8080:8080 -p 50000:50000 `
  -v jenkins_home:/var/jenkins_home `
  -v //var/run/docker.sock:/var/run/docker.sock `
  --name jenkins jenkins/jenkins:lts

  ```

Note: On Windows, use `//var/run/docker.sock` (double slash) instead of `/var/run/docker.sock`.

3. Installs tools inside jenkins container:

 ```bash
  docker exec -u root -it jenkins bash
apt update && apt install -y python3 python3-pip ansible wget unzip gnupg lsb-release
wget https://releases.hashicorp.com/terraform/1.15.1/terraform_1.15.1_linux_amd64.zip
unzip terraform_1.15.1_linux_amd64.zip && mv terraform /usr/local/bin/
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor -o /etc/apt/keyrings/trivy.gpg
echo "deb [signed-by=/etc/apt/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/trivy.list
apt update && apt install -y trivy
chmod 666 /var/run/docker.sock
exit
```
4. Access Jenkins UI at `http://localhost:8080` and complete the initial setup.

### Toolchain Installation
The following tools must be installed inside the Jenkins container to facilitate the pipeline:
* **Terraform:** v1.15.1
* **Ansible:** For remote configuration management.
* **Trivy:** For multi-stage security scanning.
* **Python3 & Pip:** Required for Ansible modules.

---

## 3. Credential Management

The following credentials must be configured in the Jenkins Global Credentials Store:

| Credential ID | Type | Description |
| :--- | :--- | :--- |
| aws-access-key | Secret Text | AWS IAM Access Key |
| aws-secret-key | Secret Text | AWS IAM Secret Key |
| dockerhub-id | Username/Password | DockerHub Registry authentication |
| ec2-ssh-key | SSH Username with private key | ec2-user + private key from devops-key.pem |

---

## 4. Component Details

### Security Scanning (Trivy)
The pipeline implements security at two gates. The first gate scans the application's `package.json` and lock files. The second gate scans the final Docker image layers to ensure no Critical or High vulnerabilities reach production.

### Infrastructure (Terraform)
Terraform manages the EC2 lifecycle. A critical configuration update was made to the `root_block_device` to increase the volume size to **20GB**, ensuring sufficient space for the Prometheus data logs and multiple Docker layers.

### Configuration Management (Ansible)
Ansible automates the "Day 1" operations on the target node:
* Installation of Docker and dependencies.
* Deployment of **Node Exporter** as a systemd service (Port 9100).
* Deployment of **Prometheus** (Port 9090).
* Deployment of **Grafana** (Port 3000).
* Deployment of the **Application Container** (Port 80).

---

## 5. Observability Stack

The monitoring stack provides real-time insights into system health:

* **Prometheus:** Configured to scrape the Node Exporter endpoint on the local host.
* **Grafana:** Connected to Prometheus as a data source.
* **Visualizations:** Utilizes Dashboard ID 1860 (Node Exporter Full) to track CPU usage, RAM utilization, and Disk I/O.



---

## 6. Maintenance and Troubleshooting

### Manual State Synchronization
In the event of a state mismatch (e.g., `InvalidGroup.Duplicate` for Security Groups), the local `terraform.tfstate` must be reconciled with the AWS environment. This is performed by executing into the Jenkins container and manually removing the state files or importing the existing resources.

### Resource Deletion
To terminate all AWS resources and prevent unnecessary billing, the following command should be executed within the terraform directory:
```bash
terraform destroy -auto-approve
```

---

## 7. Operational Flow Summary
1.  Developer updates code.
2.  Jenkins validates security and builds the image.
3.  Terraform ensures the target EC2 is in the desired state.
4.  Ansible configures the environment and starts the application.
5.  System health is verified via Grafana.
