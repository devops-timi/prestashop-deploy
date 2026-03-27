# PrestaShop Deployment on AWS — Full Documentation

## Project Overview

This project involved provisioning cloud infrastructure on **Amazon Web Services (AWS) Free Tier** and deploying a fully functional **PrestaShop e-commerce application**. The architecture separates the application server and database server across two distinct AWS services, following industry best practices for security and scalability.

**Live URL:** `http://ec2-13-220-67-36.compute-1.amazonaws.com/`

---

## Architecture Overview

```
[ Internet ]
     │
     ▼
[ EC2 Instance ]          [ RDS Instance ]
  Apache + PHP      ───►    MySQL Database
  PrestaShop App           (prestashop-db)
  (Ubuntu Linux)
```

- **Application Server:** AWS EC2 (t2.micro — Free Tier)
- **Database Server:** AWS RDS MySQL (db.t3.micro — Free Tier)
- The two services communicate internally via MySQL port 3306, secured by Security Group rules.

---

## Step 1: Launch an EC2 Instance

### 1.1 Navigate to EC2
- Log into the **AWS Management Console**
- Search for **EC2** and click **Launch Instance**

### 1.2 Configure the Instance
| Setting | Value |
|---|---|
| Name | `prestashop-server` |
| AMI | Ubuntu Server 24.04 LTS (Free Tier eligible) |
| Instance type | t2.micro (Free Tier eligible) |
| Key pair | Create new → download `.pem` file and save securely |
| Storage | 8 GB gp2 (default) |

### 1.3 Configure Security Group
Create a new Security Group with the following **Inbound Rules:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | My IP |
| HTTP | TCP | 80 | 0.0.0.0/0 |
| HTTPS | TCP | 443 | 0.0.0.0/0 |

- Click **Launch Instance**
- Wait for the Instance State to show **Running**
- Note the **Public IPv4 DNS** (e.g. `ec2-13-220-67-36.compute-1.amazonaws.com`)

---

## Step 2: Create an RDS MySQL Database

### 2.1 Navigate to RDS
- In the AWS Console, search for **RDS**
- Click **Create database**

### 2.2 Configure the Database
| Setting | Value |
|---|---|
| Creation method | Standard create |
| Engine | MySQL |
| Template | Free Tier |
| DB instance identifier | `prestashop-db` |
| Master username | `admin` |
| Master password | (your secure password) |
| DB instance class | db.t3.micro (Free Tier eligible) |
| Storage | 20 GB gp2 |
| Public access | No |

### 2.3 Configure Security Group for RDS
- Create or assign a Security Group for the RDS instance
- Add an **Inbound Rule:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| MySQL/Aurora | TCP | 3306 | EC2 Security Group (or EC2 private IP) |

> This ensures only the EC2 instance can communicate with the database — not the public internet.

- Click **Create database** and wait for the status to show **Available**
- Note the **Endpoint** under Connectivity & Security (e.g. `prestashop-db.xxxxxxxx.us-east-1.rds.amazonaws.com`)

---

## Step 3: Connect to the EC2 Instance via SSH

On your local machine, run:

```bash
chmod 400 your-key.pem
ssh -i your-key.pem ubuntu@ec2-13-220-67-36.compute-1.amazonaws.com
```

---

## Step 4: Install Apache and PHP

Once connected to the EC2 instance, run the following commands:

```bash
# Update package list
sudo apt update

# Install Apache web server
sudo apt install apache2 -y

# Install PHP and required extensions for PrestaShop
sudo apt install php php-mysql php-gd php-curl php-zip php-intl php-mbstring php-xml php-json -y

# Enable Apache mod_rewrite (required by PrestaShop)
sudo a2enmod rewrite

# Restart Apache to apply changes
sudo systemctl restart apache2
```

### 4.1 Configure Apache for PrestaShop

Edit the default Apache site configuration:

```bash
sudo nano /etc/apache2/sites-available/000-default.conf
```

Add the following block inside `<VirtualHost *:80>`:

```apache
<Directory /var/www/html>
    AllowOverride All
    Require all granted
</Directory>
```

Save and restart Apache:

```bash
sudo systemctl restart apache2
```

---

## Step 5: Download and Install PrestaShop

```bash
# Navigate to temp directory
cd /tmp

# Download PrestaShop
wget https://github.com/PrestaShop/PrestaShop/releases/download/8.1.7/prestashop_8.1.7.zip

# Install unzip utility
sudo apt install unzip -y

# Unzip the downloaded file
unzip prestashop_8.1.7.zip

# Unzip the inner PrestaShop archive to the web root
sudo unzip prestashop.zip -d /var/www/html/

# Set correct ownership and permissions
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/

# Remove the Apache default page so PrestaShop loads instead
sudo rm /var/www/html/index.html
```

---

## Step 6: Create the MySQL Database on RDS

Connect to the RDS instance from the EC2 server:

```bash
mysql -h your-rds-endpoint.rds.amazonaws.com -u admin -p
```

Inside the MySQL shell, create the database:

```sql
CREATE DATABASE `prestashop-db`;
SHOW DATABASES;
EXIT;
```

---

## Step 7: Run the PrestaShop Installation Wizard

Open a browser and navigate to:

```
http://ec2-13-220-67-36.compute-1.amazonaws.com/
```

The PrestaShop Installation Assistant will launch. Follow the steps:

### Step 7.1 — Choose Language
Select **English** and click Next.

### Step 7.2 — License Agreements
Accept the license terms and click Next.

### Step 7.3 — System Compatibility
The installer checks your server environment. If **Apache mod_rewrite** shows as missing, go back to Step 4 and re-run the `a2enmod rewrite` command, then click **Refresh information**.

### Step 7.4 — Store Information
| Field | Value |
|---|---|
| Store name | Your store name |
| Country | Your country |
| Timezone | Your timezone |
| Enable SSL | No |
| Admin email | Your email address (used to log in) |
| Admin password | A strong password (min. 8 characters) |

### Step 7.5 — Content of Your Store
Choose whether to install sample products. Recommended to select **Yes** for a demo store.

### Step 7.6 — System Configuration (Database)
| Field | Value |
|---|---|
| Database server address | Your RDS endpoint |
| Database name | `prestashop-db` |
| Database login | `admin` |
| Database password | Your RDS master password |
| Tables prefix | `ps_` |

Click **"Test your database connection now!"** — confirm you see a success message, then click **Next**.

### Step 7.7 — Store Installation
PrestaShop installs automatically. Wait for all progress bars to complete.

---

## Step 8: Post-Installation Security Step

After installation, PrestaShop blocks access to the back office until the install folder is deleted. SSH back into EC2 and run:

```bash
sudo rm -rf /var/www/html/install
```

---

## Final URLs

| Resource | URL |
|---|---|
| 🛍️ Public Store | `http://ec2-13-220-67-36.compute-1.amazonaws.com/` |
| ⚙️ Admin Back Office | `http://ec2-13-220-67-36.compute-1.amazonaws.com/admin633vjagodanyhbdkswi/` |

Log into the admin panel using the email and password set during Step 7.4.

---

## Key Configuration Decisions

### Why separate EC2 and RDS?
Hosting the application and database on the same server is a security and scalability anti-pattern. Separating them means:
- The database is not directly exposed to the internet
- Each service can be scaled independently
- A compromise of the web server does not automatically expose the database credentials at the OS level

### Why restrict RDS to port 3306 from EC2 only?
The RDS Security Group only allows inbound MySQL traffic from the EC2 instance's Security Group. This means the database cannot be reached from any other IP address on the internet, significantly reducing the attack surface.

### Why keep SSL disabled?
The EC2 public DNS URL is `http://` only. Enabling SSL without a valid certificate and domain name would break access to the store. For a production deployment, a custom domain and ACM (AWS Certificate Manager) SSL certificate would be configured.

---

## AWS Services Used

| Service | Purpose | Free Tier Limit |
|---|---|---|
| EC2 (t2.micro) | Application server | 750 hrs/month for 12 months |
| RDS (db.t3.micro) | MySQL database | 750 hrs/month for 12 months |
| Security Groups | Network access control | Free |
| EBS Storage | EC2 disk storage | 30 GB/month |
| RDS Storage | Database storage | 20 GB/month |

---

## Skills Demonstrated

- AWS EC2 provisioning and configuration
- AWS RDS MySQL setup and management
- Linux (Ubuntu) server administration
- Apache web server configuration
- PHP application deployment
- MySQL database administration
- Network security via AWS Security Groups
- Separation of application and database tiers