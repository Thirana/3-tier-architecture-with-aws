## Application Server Setup Commands

This guide provides the commands to configure the Application Tier server on an Amazon Linux 2023 EC2 instance, including MySQL client installation, code retrieval from S3, Node.js setup, and application startup.

### Installing MySQL Client

_Reference: [Installing MySQL on Amazon Linux 2023](https://dev.to/aws-builders/installing-mysql-on-amazon-linux-2023-1512)_

```bash
# Switch to the ec2-user account with sudo privileges
sudo -su ec2-user

# Download the MySQL 8.0 community release package for EL9
sudo wget https://dev.mysql.com/get/mysql80-community-release-el9-1.noarch.rpm

# Install the MySQL release package
sudo dnf install mysql80-community-release-el9-1.noarch.rpm -y

# Import the MySQL GPG key for package verification
sudo rpm --import https://repo.mysql.com/RPM-GPG-KEY-mysql-2023

# Install the MySQL client to connect to the database
sudo dnf install mysql-community-client -y

# Verify the MySQL client version
mysql --version

# Test connectivity to the RDS database (replace <RDS-Endpoint>, <username>, and provide password when prompted)
mysql -h <RDS-Endpoint> -u <username> -p
```

### Copying Application Code from S3

```bash
# Navigate to the ec2-user home directory
cd /home/ec2-user

# Copy the app-tier code from your S3 bucket (replace <YOUR-S3-BUCKET-NAME> with your bucket name)
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/app-tier app-tier --recursive

# Enter the app-tier directory
cd app-tier

# Set ownership of the app-tier directory to ec2-user
sudo chown -R ec2-user:ec2-user /home/ec2-user/app-tier

# Set appropriate permissions for the app-tier directory
sudo chmod -R 755 /home/ec2-user/app-tier
```

### Installing Node.js

```bash
# Install Node Version Manager (nvm) to manage Node.js versions
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Reload the shell configuration to use nvm
source ~/.bashrc

# Install Node.js version 16
nvm install 16

# Set Node.js version 16 as the active version
nvm use 16

# Install PM2 globally (a process manager for Node.js)
npm install -g pm2

# Install project dependencies
npm install

# Fix any vulnerabilities in dependencies
npm audit fix
```

### Starting the Application

```bash
# Start the index.js file with PM2 to run the application
pm2 start index.js

# View real-time logs (press Ctrl+C to exit)
pm2 logs

# Configure PM2 to start on system boot
pm2 startup

# Set up PM2 systemd integration for ec2-user (run this exact command)
sudo env PATH=$PATH:/home/ec2-user/.nvm/versions/node/v16.20.2/bin /home/ec2-user/.nvm/versions/node/v16.20.2/lib/node_modules/pm2/bin/pm2 startup systemd -u ec2-user --hp /home/ec2-user

# Save the current PM2 process list for boot persistence
pm2 save

# Check the application’s health endpoint
curl http://localhost:4000/health
```
