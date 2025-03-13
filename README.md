# AWS Three Tier Web Architecture

## Architecture Overview

![AWS Architecture - DrawIO](images/aws-3-tier.png)

In this architecture, a public-facing Application Load Balancer forwards client traffic to web tier EC2 instances. The web tier is running Nginx webservers that are configured to serve a React.js website and redirects our API calls to the application tier’s internal facing load balancer. The internal facing load balancer then forwards that traffic to the application tier, which is written in Node.js. The application tier manipulates data in an Aurora MySQL multi-AZ database and returns it to our web tier. Load balancing, health checks and autoscaling groups are created at each layer to maintain the availability of this architecture

## Architecture Overview

### Web Tier

- **Web Server**: Powered by `nginx` to serve static content and handle HTTP requests.
- **Frontend**: Built with `NodeJS` and React for a dynamic user interface.
- **Port**: Operates on port `80`.
- **Subnets**:
  - `10.0.1.0/24` (Availability Zone 1a)
  - `10.0.4.0/24` (Availability Zone 1b)
- **Networking**: Requires a `NAT Gateway` with an associated route table for private subnet access to the internet.

### Application Tier

- **Backend**: Uses `NodeJS` for server-side logic.
- **Database Access**: Connects to `MySQL` for data operations.
- **Port**: Runs on port `4000`.
- **Subnets**:
  - `10.0.2.0/24` (AZ 1a)
  - `10.0.5.0/24` (AZ 1b)

### Database Tier

- **Database Engine**: Utilizes `MySQL` for relational data storage.
- **Port**: Accessible on port `3306`.
- **Subnets**:
  - `10.0.3.0/24` (AZ 1a)
  - `10.0.6.0/24` (AZ 1b)

### Additional AWS Resources

- **VPC**: Configured with CIDR `10.0.0.0/16` to encompass all tiers.
- **High Availability**:
  - Resources deployed across at least two Availability Zones (AZs).
  - Uses two `Application Load Balancers`:
    - 1 external (for Web Tier)
    - 1 internal (for App Tier).
- **Fault Tolerance**:
  - `Auto Scaling` enabled for Web and App Tiers to handle traffic fluctuations.
  - Multi-AZ deployment for the Database Tier.
  - Load balancers distribute traffic efficiently.
- **Security**:
  - Ports `4000` (App Tier) and `3306` (DB Tier) are secured with restricted access.
- **Storage & Monitoring**:
  - `S3` bucket stores code for Web and App Tiers, with `IAM` roles for server access.
  - `Internet Gateway` provides public internet access to Web Tier subnets.
  - Another `S3` bucket captures VPC flow logs for auditing.
  - `CloudWatch` monitors servers and triggers `SNS` notifications to developers when needed.
- **Scaling & Alerts**:
  - `Auto Scaling Groups (ASG)` dynamically adjust server capacity.
  - `SNS` notifies developers of scaling events or issues.

## Project Development Steps

This section outlines the steps to set up the 3-tier web architecture on AWS

<details>
<summary><strong>Setting up S3 and Network Connections</strong></summary>

- **Code Storage**: Download the project code and upload the `application code` folder to a new `S3` bucket.
- **IAM Role**: Create an `IAM Role` for EC2 instances with the following policies:
  - `AmazonS3ReadOnlyAccess` (for code access).
  - `AmazonSSMManagedInstanceCore` (for Systems Manager access).
- **VPC Setup**: Create a `VPC` with the CIDR block `10.0.0.0/16`.
- **Subnets**: Create 6 subnets across two Availability Zones (AZs) using the specified CIDR blocks:
  - Web Tier: `10.0.1.0/24` (AZ 1a), `10.0.4.0/24` (AZ 1b) — enable auto-assign public IPs.
  - App Tier: `10.0.2.0/24` (AZ 1a), `10.0.5.0/24` (AZ 1b).
  - DB Tier: `10.0.3.0/24` (AZ 1a), `10.0.6.0/24` (AZ 1b).
- **VPC Flow Logs**: Set up flow logs for the VPC and store them in a new `S3` bucket.
- **Internet Gateway**: Create and attach an `Internet Gateway` to the VPC for public access.
- **Web Tier Routing**: Create a `Route Table` for Web Tier subnets with:
  - Route: `0.0.0.0/0` → `Internet Gateway`.
  - Subnet Associations: `web-tier-subnet-1` and `web-tier-subnet-2`.
- **NAT Gateways**: Create `NAT Gateways` for each Web Tier subnet:
  - One in `web-tier-subnet-1` and one in `web-tier-subnet-2`.
  - Set connectivity type to `public` and allocate an Elastic IP for each.
- **App Tier Routing (AZ 1)**: Create a `Route Table` for App Tier in AZ 1 with:
  - Route: `0.0.0.0/0` → `NAT Gateway` in `web-tier-subnet-1`.
  - Subnet Association: `App-tier-subnet-1`.
- **App Tier Routing (AZ 2)**: Create a `Route Table` for App Tier in AZ 2 with:
  - Route: `0.0.0.0/0` → `NAT Gateway` in `web-tier-subnet-2`.
  - Subnet Association: `App-tier-subnet-2`.
- **Note**: App and DB Tiers in the same AZ communicate within the VPC without additional routes.

</details>

---

<details>
<summary><strong>Creating Security Groups</strong></summary>

Configure the following security groups with specific inbound rules to ensure secure communication between tiers:

1. **`External-Load-Balancer-SG`**:
   - Rule: HTTP (port `80`) from `0.0.0.0/0` (public access).
2. **`Web-Tier-SG`**:
   - Rule: HTTP (port `80`) from `External-Load-Balancer-SG`.
3. **`Internal-Load-Balancer-SG`**:
   - Rule: HTTP (port `80`) from `Web-Tier-SG`.
4. **`App-Tier-SG`**:
   - Rule: Custom (port `4000`) from `Internal-Load-Balancer-SG`.
5. **`DB-Tier-SG`**:
   - Rule: MySQL (port `3306`) from `App-Tier-SG`.

</details>

---

<details>
<summary><strong>Setting up RDS</strong></summary>

- **RDS Subnet Group**: Create an `RDS Subnet Group`:
  - Select AZs used for `DB-tier-1` (AZ 1a) and `DB-tier-2` (AZ 1b).
  - Include subnets: `10.0.3.0/24` and `10.0.6.0/24`.
- **RDS Instance**: Set up the RDS database:
  - Creation Method: Standard, using `Aurora with MySQL`.
  - Credentials: Self-managed.
  - VPC: Use the project’s `VPC` (`10.0.0.0/16`).
  - Subnet Group: Select the `RDS Subnet Group` created above.
  - Public Access: Disabled (private access only).
  - Security Group: Attach the `DB-Tier-SG` for secure connectivity.

</details>

---

<details>
<summary><strong>Create App-Tier Server</strong></summary>

##### Creating a Testing Server to Check Functionality

- **EC2 Instance**: Launch an `EC2` instance with:
  - AMI: `Amazon Linux 2023 AMI`.
  - Instance Type: `t2.micro`.
  - Key Pair: None (use Session Manager for access).
  - Network: Select the project’s `VPC`, an App Tier subnet (`10.0.2.0/24` or `10.0.5.0/24`), and the `App-Tier-SG` security group.
  - IAM Role: Attach the previously created `IAM Role` via the instance profile.
- **Setup**: Connect to the EC2 instance using AWS Session Manager and install required packages by running the commands listed in [this file](link-to-commands-file) (available in the repo).

#### Creating an Auto-Scaling Group

After verifying functionality on the test server:

- **AMI Creation**: Create an `AMI` from the configured EC2 instance.
- **Launch Template**: Set up a `Launch Template` with:
  - AMI: Use the custom AMI created above.
  - Security Group: Select `App-Tier-SG`.
  - IAM Role: Add the correct role under advanced settings.
- **Target Group**: Create a `Target Group` with:
  - Port: `4000`.
- **Application Load Balancer**: Configure an internal `Application Load Balancer`:
  - Type: Internal (not internet-facing).
  - VPC: Select the project’s `VPC`.
  - Subnets: Use both App Tier subnets (`10.0.2.0/24`, `10.0.5.0/24`) across AZs.
  - Security Group: Select `Internal-Load-Balancer-SG`.
  - Target Group: Link to the created target group.
- **Auto Scaling Group**: Set up an `Auto Scaling Group` with:
  - Launch Template: Use the one created above.
  - VPC: Select the project’s `VPC`.
  - Subnets: Include both App Tier subnets.
  - Load Balancer: Attach to the internal load balancer and target group.
- **NGINX Configuration**: Edit the `nginx.conf` file locally to include the Internal Load Balancer’s DNS, then upload it to the `S3` bucket.

</details>

---

<details>
<summary><strong>Create Web-Tier Server</strong></summary>

- **Follow App Tier Steps**: Use a similar process as the App Tier server setup:
  - Launch an `EC2` instance with:
    - AMI: `Amazon Linux 2023 AMI`.
    - Instance Type: `t2.micro`.
    - Key Pair: None (use Session Manager).
    - Network: Select the project’s `VPC`, a Web Tier subnet (`10.0.1.0/24` or `10.0.4.0/24`), and the `Web-Tier-SG` security group.
    - IAM Role: Attach the previously created `IAM Role` via the instance profile.
  - Connect via Session Manager and install required packages using the commands in [this file](link-to-commands-file) (available in the repo).
- **Scaling**: After testing, follow the same steps as the App Tier to create an AMI, Launch Template, Target Group, Load Balancer (external this time), and Auto Scaling Group, adjusting for Web Tier specifics (e.g., port `80`, `External-Load-Balancer-SG`).

</details>
