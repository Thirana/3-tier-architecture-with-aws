## Web Server Setup Commands

This guide provides the commands to configure the Web Tier server on an Amazon Linux 2023 EC2 instance, including copying code from S3, installing Node.js for React, building the application, and setting up NGINX as the web server.

### Copying Web-Tier Code from S3

```bash
# Switch to the ec2-user account with sudo privileges
sudo -su ec2-user

# Navigate to the ec2-user home directory
cd /home/ec2-user

# Copy the web-tier code from your S3 bucket (replace <YOUR-S3-BUCKET-NAME> with your bucket name)
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/web-tier web-tier --recursive

# Set ownership of the home directory and its contents to ec2-user
sudo chown -R ec2-user:ec2-user /home/ec2-user

# Set appropriate permissions for the home directory and its contents
sudo chmod -R 755 /home/ec2-user
```

### Installing Node.js for React

```bash
# Install Node Version Manager (nvm) to manage Node.js versions
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

# Reload the shell configuration to use nvm
source ~/.bashrc

# Install Node.js version 16
nvm install 16

# Set Node.js version 16 as the active version
nvm use 16

# Navigate to the web-tier directory
cd /home/ec2-user/web-tier

# Install project dependencies for the React application
npm install

# (Optional) Fix any vulnerabilities in dependencies - uncomment if needed
# npm audit fix
```

### Building the React Application

```bash
# Navigate to the web-tier directory
cd /home/ec2-user/web-tier

# Build the React application for production (creates optimized static files for NGINX)
npm run build
```

### Installing NGINX (Web Server)

Note: Before running these commands, ensure the `nginx.conf` file includes the Internal Load Balancer DNS and is uploaded to your S3 bucket.

```bash
# Install NGINX using the yum package manager
sudo yum install nginx -y

# Navigate to the NGINX configuration directory
cd /etc/nginx

# Back up the default NGINX configuration file
sudo mv nginx.conf nginx-backup.conf

# Copy the custom NGINX configuration from your S3 bucket (replace <YOUR-S3-BUCKET-NAME> with your bucket name)
sudo aws s3 cp s3://<YOUR-S3-BUCKET-NAME>/application-code/nginx.conf .

# Set appropriate permissions for the home directory (ensures NGINX can access files)
sudo chmod -R 755 /home/ec2-user

# Restart NGINX to apply the new configuration
sudo service nginx restart

# Enable NGINX to start automatically on system boot
sudo chkconfig nginx on
```
