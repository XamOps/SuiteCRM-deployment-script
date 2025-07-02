# SuiteCRM 8 Automated Deployment on AWS EC2 (Private Subnet via ALB)

This repository contains a Bash script to automate the installation and configuration of SuiteCRM 8.x on an Ubuntu 24.04 EC2 instance located in a private subnet, accessible via an AWS Application Load Balancer (ALB).

## Table of Contents
- [Project Overview](#project-overview)
- [Features](#features)
- [Prerequisites](#prerequisites)
  - [AWS Infrastructure](#aws-infrastructure)
  - [EC2 Instance & Access](#ec2-instance--access)
- [Setup & Usage](#setup--usage)
  - [1. Configure the Script](#1-configure-the-script)
  - [2. Perform Manual Database Setup](#2-perform-manual-database-setup)
  - [3. Run the Deployment Script](#3-run-the-deployment-script)
  - [4. Final Manual Configuration Checks](#4-final-manual-configuration-checks)
- [Important Notes](#important-notes)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

## Project Overview

This script streamlines the deployment of a robust and secure SuiteCRM 8 instance on AWS. By leveraging private subnets for the EC2 instance and an Application Load Balancer for public access, it ensures that your CRM backend is not directly exposed to the internet, enhancing security.

The script automates the installation of the necessary LAMP (Linux, Apache, MySQL/MariaDB, PHP) stack components, downloads and configures SuiteCRM, and performs post-installation setup steps.

## Features

* **Automated LAMP Stack Installation:** Installs Apache2, PHP 8.3 (with all required extensions), and MariaDB Server.
* **SuiteCRM Download & Extraction:** Fetches the specified SuiteCRM 8.x version and prepares its directory structure.
* **Apache Configuration:** Sets up a dedicated Apache Virtual Host for SuiteCRM, directing traffic to its secure `public` directory.
* **SuiteCRM CLI Installation:** Executes the official SuiteCRM command-line installer for a headless deployment.
* **Permission Management:** Configures appropriate file and directory permissions for SuiteCRM.
* **Error Handling:** Includes basic checks and exits on critical errors.

## Prerequisites

### AWS Infrastructure

Before running this script, you **must** have the following AWS infrastructure already provisioned:

* **VPC:** A Virtual Private Cloud setup with:
    * **Public Subnets:** At least two for the ALB's high availability.
    * **Private Subnets:** At least one (preferably two for redundancy) where your EC2 instance will reside.
* **Internet Gateway (IGW):** Attached to your VPC.
* **NAT Gateway:** Deployed in a public subnet, configured to allow outbound internet access from your private subnets.
* **Route Tables:** Configured correctly to direct traffic through the IGW and NAT Gateway.
* **Security Groups:**
    * **ALB Security Group:** Allows inbound HTTP (port 80) from `0.0.0.0/0`.
    * **EC2 Instance Security Group:** Allows inbound HTTP (port 80) from the **ALB Security Group** only. Also allows SSH (port 22) from your administrative IP address range or a secure bastion host/Systems Manager.
* **EC2 Instance:** An **Ubuntu 24.04 LTS** instance launched into a **private subnet**. Ensure it has an IAM role with sufficient permissions if using Systems Manager, or an associated key pair for direct SSH.
* **Application Load Balancer (ALB):** An active ALB in your public subnets, configured with an HTTP listener (port 80) and a target group pointing to your private EC2 instance. The ALB must have a DNS name (e.g., `your-alb-name.region.elb.amazonaws.com`).
* **SSH Access:** You must have SSH access to your private EC2 instance (e.g., via EC2 Connect Endpoint, a bastion host, or SSH key pair).

### EC2 Instance & Access

* **Operating System:** Ubuntu 24.04 LTS.
* **User:** You must be logged in as a user with `sudo` privileges (e.g., `ubuntu` user).
* **Internet Connectivity:** The EC2 instance (despite being in a private subnet) **must** have outbound internet access via a NAT Gateway to download packages and SuiteCRM files.

## Setup & Usage

Follow these steps carefully to deploy SuiteCRM.

### 1. Configure the Script

1.  **Clone the Repository:**
    ```bash
    git clone [https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY_NAME.git](https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPOSITORY_NAME.git)
    cd YOUR_REPOSITORY_NAME
    ```
2.  **Make the Script Executable:**
    ```bash
    chmod +x deploy_suitecrm.sh
    ```
3.  **Edit the Configuration Variables:**
    Open `deploy_suitecrm.sh` in a text editor (e.g., `nano deploy_suitecrm.sh`).
    ```bash
    nano deploy_suitecrm.sh
    ```
    Modify the variables in the `--- SCRIPT CONFIGURATION ---` section:
    * `ALB_DNS_NAME`: Replace with your ALB's actual DNS name (e.g., `"http://your-alb-name.region.elb.amazonaws.com"`).
    * `DB_ROOT_PASSWORD`: **Define a strong password** for your MariaDB `root` user. You will use this manually. Example: `"MyS3cUr3DBR00tP@ss!"`
    * `SUITECRM_DB_PASSWORD`: **Define a strong password** for the `suitecrm_user` database user. Example: `"CrmDbUserP@ss#1"`
    * `SUITECRM_ADMIN_PASSWORD`: **Define a strong password** for the SuiteCRM application's `admin` user. Example: `"CrmAdm1nP@ss$2"`
    * `SUITECRM_VERSION`: (e.g., "8.8.0"). Update if deploying a different version.
    * `SUITECRM_ZIP_URL` and `SUITECRM_ZIP_FILENAME`: **Crucially, verify this URL** by visiting `https://suitecrm.com/download/` and getting the direct link for the `SuiteCRM-8.x.x.zip` file. The provided GitHub URL is usually reliable for specific versions, but always confirm.

    **Save the changes and exit the editor.**

### 2. Perform Manual Database Setup

The script will pause for these steps. You **must** execute these manually as instructed.

1.  **Secure MariaDB Installation:**
    When prompted by the script, run:
    ```bash
    sudo mysql_secure_installation
    ```
    * **Current root password:** If this is a fresh install, press `Enter` (no password).
    * **Set new root password:** Choose `Y` and enter the `DB_ROOT_PASSWORD` you defined in the script.
    * **Remove anonymous users:** `Y`
    * **Disallow root login remotely:** `Y` (Highly Recommended for production)
    * **Remove test database:** `Y`
    * **Reload privilege tables:** `Y`

2.  **Create SuiteCRM Database and User:**
    After securing MariaDB, the script will prompt you again.
    Log into MariaDB as root:
    ```bash
    sudo mariadb -u root -p
    ```
    *Enter your `DB_ROOT_PASSWORD` when prompted.*

    Then, **copy and paste the exact SQL commands provided by the script** into the MariaDB monitor:
    ```sql
    CREATE DATABASE suitecrm_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
    CREATE USER 'suitecrm_user'@'localhost' IDENTIFIED BY 'YOUR_SUITECRM_DB_PASSWORD_HERE';
    GRANT ALL PRIVILEGES ON suitecrm_db.* TO 'suitecrm_user'@'localhost';
    FLUSH PRIVILEGES;
    EXIT;
    ```
    *(The script will display these exact commands for you with your chosen passwords.)*

### 3. Run the Deployment Script

Once you have completed the manual database setup steps, proceed to run the script.

```bash
sudo ./deploy_suitecrm.sh
