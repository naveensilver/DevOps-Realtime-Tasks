# End-to-End Step-by-Step Guide: Automating AWS AMI Creation Using HashiCorp Packer

In this guide, we will go over how to automate the creation of an AWS AMI (Amazon Machine Image) with Docker, Node Exporter, and an Ansible admin user using HashiCorp Packer. We'll also provide detailed explanations for each step, along with the necessary changes and validations you need to perform to ensure the automation is successful.


Let’s integrate the **AWS CLI configuration** and your step-by-step **Packer template setup** into a comprehensive guide for creating a production-ready **Amazon Machine Image (AMI)** using **Packer**. This includes AWS authentication, preparing the environment, configuring the template, provisioning the instance, and best practices for managing SSH keys and credentials securely.

---

### **Step-by-Step Implementation for Production (Including AWS CLI Configuration)**

Here’s the **full process**, including **AWS CLI configuration** to ensure that you can use **Packer** to create an AMI, provision it, and securely authenticate with AWS.

---

### **1. Install Prerequisites**

Ensure you have the following installed on your local machine or CI/CD environment:

1. **AWS CLI**: For interacting with AWS services.
2. **Packer**: For automating the AMI creation process.
3. **Terraform** (optional, but useful for infrastructure management if needed).

---

### **2. Configure AWS CLI**

To authenticate with **AWS** using the **AWS CLI**, you need to configure your credentials.

#### **Step 1: Install AWS CLI (if not installed already)**

For installation instructions, refer to the [AWS CLI installation guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html).

After installation, run the following command to configure your credentials:

```bash
aws configure
```

This will prompt you for the following details:

- **AWS Access Key ID**: This is your AWS access key, which you can create via **IAM (Identity and Access Management)** in AWS Console.
- **AWS Secret Access Key**: The secret key associated with your AWS access key.
- **Default region name**: The region where your EC2 instances and AMIs will be created. For example, `us-east-1`.
- **Default output format**: You can choose `json` or `text` depending on your preference.

Example:

```bash
AWS Access Key ID [None]: XXXXXXXXXXXXXXXXXXXX
AWS Secret Access Key [None]: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Default region name [None]: us-east-1
Default output format [None]: json
```

Make sure that the **AWS IAM user** you are using has the appropriate permissions to manage EC2, AMIs, and other services you'll use. The following permissions are typically required:

- `AmazonEC2FullAccess`
- `IAMFullAccess` (for managing key pairs and access)
- `AmazonS3FullAccess` (if using S3 for file uploads or storage)

---

### **3. Prepare and Configure Packer Template**

Once the **AWS CLI** is configured, you can proceed with the **Packer setup**.

You need to configure the following files:

- **`packer.json`** (Packer template)
- **`vars.json`** (Variables file containing sensitive and configurable values)
- **`docker.service`** (Docker systemd service file)
- **`node_exporter.service`** (Node Exporter systemd service file)

---

#### **1. Configure `packer.json`**

In your `packer.json`, define the following key settings for the **AWS EC2 instance**:

- **AWS region**: Where the AMI will be created.
- **Source AMI**: The base AMI (e.g., Ubuntu, Amazon Linux).
- **Instance Type**: The EC2 instance type for building the image.
- **VPC and Subnet IDs**: These are the network settings for your EC2 instance.
- **SSH Keypair**: Used to connect to the EC2 instance for provisioning.

Here is an example `packer.json` template:

```json
{
  "_comment": "Packer template for creating an AMI with Docker, Nginx, Node Exporter, and Ansible",
  
  "variables": {
    "region": "us-east-1",
    "source_ami": "ami-0866a3c8686eaeeba", // Base AMI (Ubuntu)
    "instance_type": "t2.micro",
    "vpc_id": "vpc-036e5c5d11bdf83de",
    "subnet_id": "subnet-05597e96c163e70fd",
    "ssh_keypair_name": "your-ssh-key-name", // Ensure you have an existing SSH key pair in AWS
    "ssh_private_key_file": "{{env `SSH_PRIVATE_KEY_FILE`}}", // Reference private key path from environment variable
    "ssh_public_key": "{{env `SSH_PUBLIC_KEY`}}" // Reference public key from environment variable
  },

  "builders": [
    {
      "type": "amazon-ebs",
      "region": "{{user `region`}}",
      "source_ami": "{{user `source_ami`}}",
      "instance_type": "{{user `instance_type`}}",
      "ssh_username": "ubuntu",
      "ami_name": "DevSecOps-Ansible-Image-{{isotime | clean_resource_name}}",
      "vpc_id": "{{user `vpc_id`}}",
      "subnet_id": "{{user `subnet_id`}}",
      "ssh_keypair_name": "{{user `ssh_keypair_name`}}",
      "ssh_private_key_file": "{{user `ssh_private_key_file`}}",
      "tags": {
        "Name": "DevSecOps-Ansible-Image-{{isotime | clean_resource_name}}"
      }
    }
  ],

  "provisioners": [
    {
      "type": "shell",
      "inline": [
        "sudo useradd -m ansibleadmin --shell /bin/bash",
        "sudo mkdir -p /home/ansibleadmin/.ssh",
        "sudo chown -R ansibleadmin /home/ansibleadmin/",
        "sudo touch /home/ansibleadmin/.ssh/authorized_keys",
        "sudo usermod -aG sudo ansibleadmin",
        "echo '{{user `ssh_public_key`}}' | sudo tee /home/ansibleadmin/.ssh/authorized_keys"
      ]
    },
    {
      "type": "shell",
      "inline": [
        "sudo apt update -y",
        "curl https://get.docker.com | bash"
      ]
    },
    {
      "type": "file",
      "source": "docker.service",
      "destination": "/tmp/docker.service"
    },
    {
      "type": "shell",
      "inline": [
        "sudo cp /tmp/docker.service /lib/systemd/system/docker.service",
        "sudo systemctl daemon-reload",
        "sudo service docker restart"
      ]
    },
    {
      "type": "shell",
      "inline": [
        "sudo useradd --no-create-home --shell /bin/false node_exporter",
        "wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz",
        "tar xvf node_exporter-1.3.1.linux-amd64.tar.gz",
        "sudo cp node_exporter-1.3.1.linux-amd64/node_exporter /usr/local/bin/",
        "sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter"
      ]
    },
    {
      "type": "file",
      "source": "node_exporter.service",
      "destination": "/tmp/node_exporter.service"
    },
    {
      "type": "shell",
      "inline": [
        "sudo cp /tmp/node_exporter.service /etc/systemd/system/node_exporter.service",
        "sudo systemctl enable node_exporter",
        "rm -rf node_exporter*"
      ]
    }
  ]
}
```

---

#### **2. Configure `vars.json`**

`vars.json` contains the configuration variables that Packer will use for the build.

```json
{
  "region": "us-east-1",
  "source_ami": "ami-0866a3c8686eaeeba",
  "instance_type": "t2.micro",
  "vpc_id": "vpc-036e5c5d11bdf83de",
  "subnet_id": "subnet-05597e96c163e70fd",
  "ssh_keypair_name": "your-ssh-key-name",  // Replace with your SSH key pair
  "ssh_private_key_file": "{{env `SSH_PRIVATE_KEY_FILE`}}",  // Private key path (environment variable)
  "ssh_public_key": "{{env `SSH_PUBLIC_KEY`}}"  // Public key for SSH (environment variable)
}
```

**Note**: For security reasons, do not hardcode your private or public keys in the template. Use environment variables to securely pass them at runtime.


#### **Docker Service File (docker.service)**

The `docker.service` file configures Docker to start with systemd, listening on both the Unix socket and a TCP port (`2375`).

```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket

[Service]
Type=notify
ExecStart=/usr/bin/dockerd -H unix:// -H tcp://0.0.0.0:2375 -H fd:// --containerd=/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target

```
- This service ensures Docker runs on startup, listens on TCP port 2375 (for remote access), and uses systemd's restart policies.

#### **4.Node Exporter Service File (node_exporter.service)**

The `node_exporter.service` file configures the Prometheus Node Exporter to expose system metrics.

```
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
- It runs the Node Exporter under the `node_exporter` user for security purposes.

---

### **4. Set Environment Variables**

Before running Packer, set environment variables for your SSH keys to avoid exposing them in files.

For **Linux/macOS**:

```bash
export SSH_PRIVATE_KEY_FILE="/path/to/your/private/key.pem"
export SSH_PUBLIC_KEY="your-public-key-string-here"
```

For **Windows** (in PowerShell):

```powershell
$Env:SSH_PRIVATE_KEY_FILE = "C:\path\to\your\private\key.pem"
$Env:SSH_PUBLIC_KEY = "ssh-rsa AAAAB3Nza...your-public-key-here"
```

---

### **5. Validate and Build the Image**

#### **Step 1: Validate the Template**

Before running the build, validate your Packer template to ensure everything is correct:

```bash
packer validate --var-file=vars.json packer.json
```

If the validation is successful, you will see a confirmation message.

#### **Step 

2: Build the AMI**

Once validated, run the build command:

```bash
packer build --var-file=vars.json packer.json
```

This will:

- Launch an EC2 instance using the base AMI.
- Provision the instance with Docker, Node Exporter, Ansible, etc.
- Package the instance into a new custom AMI.

---

### **6. Use the Newly Created AMI**

Once the build is complete, Packer will output the **AMI ID**. You can now use this custom AMI to launch new EC2 instances with the preconfigured software.

---

### **7. Best Practices for Production**

1. **Secure your SSH keys**: Use environment variables, AWS Secrets Manager, or HashiCorp Vault for managing sensitive data.
2. **Validate your template**: Always validate the template before production runs to avoid errors.
3. **Automate with CI/CD**: Integrate Packer into your CI/CD pipeline for automated AMI creation.
4. **Test the AMI**: Before deploying to production, launch an EC2 instance from your custom AMI to ensure everything works as expected.

---

### **Conclusion**

By following these steps, you've configured a **production-ready Packer template** to create a custom AMI with Docker, Node Exporter, and Ansible. You’ve also learned how to securely manage **AWS credentials** and **SSH keys**, validate the template, and automate the AMI creation process with **Packer**.