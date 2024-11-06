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

---

# Automate Above Process using CI/CD 

To automate the process of building the AMI with **Packer** in a CI/CD pipeline, we can use **GitHub Actions**. GitHub Actions allows you to define workflows in YAML files and trigger them automatically based on events like commits or pull requests. We can integrate the process of validating and building the AMI using **Packer** inside GitHub Actions.

### **Steps to Automate Packer in GitHub Actions**

Let's break it down step-by-step:

---

### **1. Set Up AWS Credentials as GitHub Secrets**

Before setting up the GitHub Actions workflow, ensure that your AWS credentials are securely stored in GitHub Secrets to allow the workflow to authenticate with AWS.

1. Go to your GitHub repository.
2. Click on **Settings** > **Secrets and Variables** > **Actions** > **New repository secret**.
3. Add the following secrets:
   - `AWS_ACCESS_KEY_ID`: Your AWS Access Key.
   - `AWS_SECRET_ACCESS_KEY`: Your AWS Secret Key.
   - `AWS_REGION`: The region where you want to build your AMI (e.g., `us-east-1`).
   - `SSH_PRIVATE_KEY_FILE`: Path to your private key (or if you want to pass it as an environment variable).
   - `SSH_PUBLIC_KEY`: Your SSH public key for the EC2 instance.

---

### **2. Create the GitHub Actions Workflow File**

In your GitHub repository, create the GitHub Actions workflow file inside the `.github/workflows` directory. If this directory doesn't exist, create it.

For example, create a new file: `.github/workflows/packer.yml`.

Here's an example GitHub Actions workflow configuration:

```yaml
name: Build AMI with Packer

on:
  push:
    branches:
      - main  # Trigger this workflow on push to the main branch
  pull_request:
    branches:
      - main  # Trigger this workflow on PR to the main branch

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    # Step 1: Check out the code
    - name: Checkout Repository
      uses: actions/checkout@v2

    # Step 2: Set up AWS credentials using GitHub Secrets
    - name: Set up AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    # Step 3: Set up environment variables for SSH keys
    - name: Set up SSH keys
      run: |
        echo "export SSH_PRIVATE_KEY_FILE=${{ secrets.SSH_PRIVATE_KEY_FILE }}" >> $GITHUB_ENV
        echo "export SSH_PUBLIC_KEY=${{ secrets.SSH_PUBLIC_KEY }}" >> $GITHUB_ENV

    # Step 4: Install Packer
    - name: Install Packer
      run: |
        curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
        sudo apt update && sudo apt install packer

    # Step 5: Validate the Packer template
    - name: Validate Packer template
      run: |
        packer validate --var-file=vars.json packer.json

    # Step 6: Build the AMI using Packer
    - name: Build AMI with Packer
      run: |
        packer build --var-file=vars.json packer.json

    # Step 7: Clean up any resources or temporary files (optional)
    - name: Clean up
      run: |
        rm -rf packer_cache  # Remove any unnecessary files
```

### **Explanation of the Workflow:**

1. **Trigger**: 
   - The workflow triggers on **push** or **pull request** events to the `main` branch. You can modify this based on your requirements.

2. **Checkout Code**:
   - This step checks out the code from the GitHub repository so that we can work with the Packer files and configuration.

3. **Set up AWS Credentials**:
   - The action `aws-actions/configure-aws-credentials@v1` sets up your AWS credentials using the GitHub secrets you stored. This allows Packer to interact with AWS without hardcoding your credentials.

4. **Set up SSH Keys**:
   - Here, we use GitHub Secrets to set up the **SSH_PRIVATE_KEY_FILE** and **SSH_PUBLIC_KEY** as environment variables. These are passed to Packer to use during the AMI creation process.

5. **Install Packer**:
   - Packer is installed on the runner using the package manager. This step is necessary because GitHub runners don't come with Packer pre-installed.

6. **Validate Packer Template**:
   - The `packer validate` command ensures that your Packer template (`packer.json`) is valid before proceeding to build the AMI. This helps catch errors early.

7. **Build AMI with Packer**:
   - The `packer build` command actually builds the AMI, using the configuration from `packer.json` and the variables defined in `vars.json`. The result will be an AMI ID that can be used in your infrastructure.

8. **Clean up (Optional)**:
   - This step is optional but useful for cleaning up any temporary files or resources that might have been created during the build process.

---

### **3. Store the Necessary Files in Your GitHub Repository**

You should have the following files in your repository:

- **`packer.json`**: The main Packer template to build the AMI.
- **`vars.json`**: The variables file that contains your configuration settings (e.g., VPC, Subnet, instance type).
- **`docker.service`**: The systemd service file for Docker.
- **`node_exporter.service`**: The systemd service file for Node Exporter.

These files should be in the root directory or a folder like `/packer` inside your repository, depending on your organization.

---

### **4. Testing the Workflow**

Once everything is set up, push your changes to GitHub or create a pull request to trigger the workflow.

```bash
git add .
git commit -m "Setup Packer CI/CD Pipeline with GitHub Actions"
git push origin main
```

GitHub will automatically trigger the action and start the AMI build process as per the workflow file. You can view the logs of the job in the **Actions** tab of your GitHub repository.

---

### **5. Monitor the Build Process**

1. Navigate to the **Actions** tab in your GitHub repository.
2. Select the latest workflow run and monitor the logs to check for any errors or confirmation that the AMI build was successful.
3. If the build is successful, the AMI will be created in the specified AWS region. The AMI ID will be output in the logs of the build step.

---

### **6. Use the Custom AMI**

After the workflow completes, you can find the AMI ID from the **build logs** and use it to launch new EC2 instances. You can also modify the workflow to output the AMI ID as part of the job summary or store it in an artifact.

---

### **Additional Enhancements and Tips**

- **Versioning**: Use versioning for your AMIs, either by including a timestamp in the AMI name or by using unique identifiers.
- **Notifications**: You can integrate notifications (e.g., Slack, email) when the build is successful or fails.
- **Testing**: Create a post-build step to test the AMI (e.g., run a smoke test in the instance created from the AMI).

---

### **Conclusion**

By following these steps, you can automate the entire process of building and managing your custom AWS AMIs using **Packer** in **GitHub Actions**. This approach ensures that your infrastructure is reproducible, consistent, and integrated into your CI/CD pipeline.