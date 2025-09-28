# AWS Helper Scripts

A collection of bash functions for managing AWS resources, including EC2 instances, RDS databases, and SSM sessions.

## Setup

1. Source the configuration file in your shell:
   ```bash
   source .aws-config
   ```

2. Add to your shell profile (`.bashrc`, `.zshrc`) for persistent access:
   ```bash
   echo "source $(pwd)/.aws-config" >> ~/.zshrc
   ```

## Prerequisites

- AWS CLI installed and configured
- AWS Session Manager plugin
- `jq` command-line JSON processor
- Appropriate AWS IAM permissions for EC2, SSM, and Secrets Manager

## Quick Command Reference

| Command | Description |
|---------|-------------|
| `aws-profile` | Switch AWS profiles or list available profiles |
| `aws-ec2-list` | List all EC2 instances with details |
| `aws-asg-refresh` | Start instance refresh for an Auto Scaling Group |
| `aws-rds-info` | Retrieve RDS credentials from Secrets Manager |
| `aws-ssh` | SSH into EC2 instance via Session Manager |
| `aws-scp` | Transfer files to/from EC2 instances via SSH over Session Manager |
| `aws-scp-ssm` | Transfer files to/from EC2 instances via SSM commands (slower) |
| `aws-db-fwdport` | Port forward to RDS through EC2 instance via SSM |

**Note:** All commands automatically display the current AWS profile and prompt you to select one if none is set.

---

## Command Details

### `aws-profile`

Manage AWS CLI profiles for different environments.

**Usage:**
```bash
aws-profile [profile-name]
```

**Parameters:**
- `profile-name` (optional): Name of AWS profile to switch to

**Examples:**
```bash
# Show current profile and list available profiles
aws-profile

# Switch to production profile
aws-profile prod

# Switch to development profile
aws-profile dev
```

**Output:**
- Without parameters: Shows current profile and lists all available profiles
- With profile name: Switches to specified profile and confirms the change

---

### `aws-ec2-list`

Display all EC2 instances in a formatted table showing instance IDs, names, and states.

**Usage:**
```bash
aws-ec2-list
```

**Parameters:**
None

**Example:**
```bash
aws-ec2-list
```

**Output:**
```
---------------------------------------------------------------------------
|                           DescribeInstances                            |
+------------------+------------------+------------------+
|  i-0123456789abcdef0 |  my-app-server |     running      |
|  i-0123456789abcdef0 |  web-server         |     stopped      |
+------------------+------------------+------------------+
```

---

### `aws-asg-refresh`

Start an instance refresh for an Auto Scaling Group with zero downtime tolerance.

**Usage:**
```bash
aws-asg-refresh
```

**Parameters:**
None

**Interactive Flow:**
1. Lists all Auto Scaling Groups with their current capacity settings
2. Prompts for ASG selection
3. Shows warning about downtime (MinHealthyPercentage=0)
4. Asks for confirmation before proceeding
5. Starts instance refresh with aggressive settings

**Example:**
```bash
aws-asg-refresh
```

**Sample Output:**
```
Fetching Auto Scaling Groups...
Available Auto Scaling Groups:
=============================
1) my-app-asg (Desired: 2, Min: 1, Max: 4)
2) web-server-asg (Desired: 3, Min: 2, Max: 6)

Select ASG number (1-2): 1
Selected ASG: my-app-asg
Starting instance refresh with MinHealthyPercentage=0, InstanceWarmup=0...
WARNING: This will cause downtime as MinHealthyPercentage=0
Continue? (y/N): y
{
    "InstanceRefreshId": "08b91cf7-8fa6-48af-84a1-6d659cae8b7a"
}
Instance refresh started successfully for my-app-asg
```

**Configuration Details:**
- `MinHealthyPercentage=0`: Allows all instances to be replaced simultaneously (causes downtime)
- `InstanceWarmup=0`: No warmup period, instances are considered healthy immediately

**Warning:**
This function uses aggressive settings that **will cause downtime**. Use only when downtime is acceptable, such as during maintenance windows.

**Prerequisites:**
- IAM permissions for `autoscaling:DescribeAutoScalingGroups` and `autoscaling:StartInstanceRefresh`

---

### `aws-rds-info`

Retrieve RDS database connection information from AWS Secrets Manager.

**Usage:**
```bash
aws-rds-info
```

**Parameters:**
None

**Example:**
```bash
aws-rds-info
```

**Output:**
```json
{
  "username": "admin",
  "password": "secret123",
  "engine": "mysql",
  "host": "my-rds-cluster.cj0example123.ap-southeast-5.rds.amazonaws.com",
  "port": 3306,
  "dbname": "production"
}
```

**Prerequisites:**
- Set `AWS_RDS_SECRET_ID` environment variable to your secret path

**Notes:**
- Fetches from secret ID specified in `AWS_RDS_SECRET_ID` environment variable
- Returns formatted JSON with database credentials

---

### `aws-ssh`

Connect to EC2 instances via SSH using AWS Session Manager.

**Usage:**
```bash
aws-ssh
```

**Parameters:**
None

**Interactive Flow:**
1. Lists all running EC2 instances
2. Prompts for instance selection
3. Establishes SSH connection via Session Manager

**Example:**
```bash
aws-ssh
```

**Sample Output:**
```
Fetching running EC2 instances...
Available running EC2 instances:
=====================================
1) i-0123456789abcdef0 - my-app-server - Private: 10.0.10.46

Select instance number (1-1): 1
Connecting to instance: i-0123456789abcdef0
Starting SSH session via AWS Session Manager...
```

**Benefits:**
- Works through private networks (no public IP required)
- Uses IAM permissions instead of SSH keys
- Auditable through CloudTrail

---

### `aws-scp`

Transfer files to and from EC2 instances using SSH over Session Manager (recommended method).

**Prerequisites:**
Before using this function, you need to set up SSH key access on your EC2 instances:

1. **Connect to instance via Session Manager:**
   ```bash
   aws-ssh  # Use our helper to connect
   ```

2. **Switch to the ubuntu/ec2-user account:**
   ```bash
   sudo su - ubuntu  # or ec2-user depending on AMI
   ```

3. **Add your SSH public key:**
   ```bash
   mkdir -p ~/.ssh
   echo "your-ssh-public-key-here" >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   chmod 700 ~/.ssh
   ```

4. **Optional: Configure your local SSH config (permanent setup):**
   Add this to your `~/.ssh/config`:
   ```
   Host i-* mi-*
       ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
       User ubuntu
       StrictHostKeyChecking no
       UserKnownHostsFile /dev/null
   ```

**Usage:**
```bash
aws-scp <source> <destination> [username]
```

**Parameters:**
- `source`: Local or remote file path (relative/absolute for local, absolute for remote)
- `destination`: Local or remote file path (relative/absolute for local, absolute for remote)
- `username`: SSH username (optional, defaults to `ubuntu`)

**Examples:**

**Upload files:**
```bash
aws-scp ./config.json /opt/app/config/         # Upload relative local file
aws-scp /home/user/file.txt /home/ubuntu/      # Upload absolute local file
aws-scp ./archive.tar.gz /tmp/                 # Upload to /tmp/archive.tar.gz
```

**Download files:**
```bash
aws-scp /var/log/app.log ./logs/               # Download to relative local path
aws-scp /etc/nginx/nginx.conf /home/user/      # Download to absolute local path
```

**Custom username:**
```bash
aws-scp ./file.txt /home/ec2-user/ ec2-user    # Use ec2-user instead of ubuntu
```

**Sample Output:**
```bash
$ aws-scp ./deploy.sh /home/ubuntu/
Fetching running EC2 instances...
Available running EC2 instances:
=====================================
1) i-0123456789abcdef0 - my-app-server - Private: 10.0.10.46

Select instance number (1-1): 1
Selected instance: i-0123456789abcdef0
Using SSH user: ubuntu

Setting up SSH proxy via Session Manager...
Uploading /Users/user/deploy.sh to i-0123456789abcdef0:/home/ubuntu/deploy.sh
deploy.sh                    100%  1247    15.2KB/s   00:00
Transfer completed successfully.
```

**Benefits:**
- **Fast transfers** - uses native SCP over SSH tunnel
- **Supports large files** - no encoding overhead
- **Works with directories** - `scp -r` functionality
- **Progress indication** - shows transfer speed and progress
- **Auto-cleanup** - temporary SSH config is cleaned up automatically

**Tips:**
- For large files, compress first: `tar czf archive.tar.gz folder/ && aws-scp archive.tar.gz /tmp/`
- Transfer is slow over Session Manager, so compress when possible
- SSH key setup is one-time per instance

---

### `aws-scp-ssm`

Transfer files to and from EC2 instances using SSM commands (fallback method when SSH keys are not available).

**Usage:**
```bash
aws-scp-ssm <source> <destination>
```

**Parameters:**
- `source`: Local or remote file path (relative/absolute for local, absolute for remote)
- `destination`: Local or remote file path (relative/absolute for local, absolute for remote)

**Examples:**
```bash
aws-scp-ssm ./config.json /opt/app/config/     # Upload relative local file
aws-scp-ssm /var/log/app.log ./logs/           # Download to relative local path
```

**Notes:**
- **Slower than aws-scp** - uses base64 encoding through SSM commands
- **No SSH keys required** - works with just SSM permissions
- **Limited file size** - may fail with very large files due to SSM command limits
- **Use aws-scp instead** when SSH keys are available

---

### `aws-db-fwdport`

Create a port forwarding tunnel to RDS database through an EC2 instance via AWS Session Manager.

**Usage:**
```bash
aws-db-fwdport [local-port] [rds-endpoint]
```

**Parameters:**
- `local-port` (optional): Local port to bind to (default: 33060)
- `rds-endpoint` (optional): RDS endpoint hostname (auto-fetched if not provided)

**Examples:**

**Auto-fetch RDS endpoint with default local port:**
```bash
aws-db-fwdport
# Connects to: auto-fetched RDS endpoint
# Port mapping: remote 3306 → local 33060
```

**Auto-fetch RDS endpoint with custom local port:**
```bash
aws-db-fwdport 3307
# Connects to: auto-fetched RDS endpoint
# Port mapping: remote 3306 → local 3307
```

**Manual RDS endpoint with default local port:**
```bash
aws-db-fwdport "" my-rds-cluster.amazonaws.com
# Connects to: my-rds-cluster.amazonaws.com
# Port mapping: remote 3306 → local 33060
```

**Manual RDS endpoint with custom local port:**
```bash
aws-db-fwdport 3307 my-rds-cluster.amazonaws.com
# Connects to: my-rds-cluster.amazonaws.com
# Port mapping: remote 3306 → local 3307
```

**Prerequisites:**
- Set `AWS_RDS_SECRET_ID` environment variable (for auto-fetch mode)
- EC2 instance with SSM agent enabled

**Interactive Flow:**
1. Fetches RDS endpoint from Secrets Manager (if not provided and `AWS_RDS_SECRET_ID` is set)
2. Lists running EC2 instances for proxy selection
3. Establishes port forwarding tunnel via AWS Session Manager
4. Keeps session active until terminated

**Sample Output:**
```bash
$ aws-db-fwdport 3307
Fetching RDS endpoint from Secrets Manager...
Found RDS endpoint: my-rds-cluster.cj0example123.ap-southeast-5.rds.amazonaws.com
Fetching running EC2 instances...
Available running EC2 instances:
=====================================
1) i-0123456789abcdef0 - my-app-server - Private: 10.0.10.46

Select instance number (1-1): 1
Starting port forwarding session...
Instance: i-0123456789abcdef0
RDS Host: my-rds-cluster.cj0example123.ap-southeast-5.rds.amazonaws.com
Remote Port: 3306 -> Local Port: 3307

Starting session with SessionId: botocore-session-123456789
Port 3307 opened for sessionId botocore-session-123456789.
Waiting for connections...
```

**Usage After Port Forwarding:**
Once the tunnel is established, connect to your database using:
```bash
mysql -h 127.0.0.1 -P 3307 -u username -p
```

**Notes:**
- Remote port is always 3306 (standard MySQL/MariaDB)
- Uses AWS Session Manager for secure tunneling (no SSH keys required)
- RDS endpoint auto-fetched from secret specified in `AWS_RDS_SECRET_ID`
- Automatically strips port from RDS endpoint if present
- Session remains active until manually terminated (Ctrl+C)

---

## Environment Variables

### Automatically Set Variables

- `AWS_DEFAULT_REGION`: ap-southeast-5
- `AWS_REGION`: ap-southeast-5
- `AWS_PROFILE`: Managed by `aws-profile` function

### Required Configuration Variables

- `AWS_RDS_SECRET_ID`: Path to your RDS credentials in AWS Secrets Manager

**Setup Example:**
```bash
# Add to your .bashrc/.zshrc or .aws-config
export AWS_RDS_SECRET_ID="myapp/database/password"
```

**Usage:**
- Used by `aws-rds-info` to fetch RDS credentials
- Used by `aws-db-fwdport` to auto-fetch RDS endpoint when not specified manually

## Troubleshooting

**Common Issues:**

1. **"No running EC2 instances found"**
   - Check AWS profile and region settings
   - Verify EC2 instances are in running state
   - Ensure proper IAM permissions for EC2 DescribeInstances

2. **"Could not fetch RDS endpoint from Secrets Manager"**
   - Set `AWS_RDS_SECRET_ID` environment variable
   - Verify secret exists at the specified path
   - Check IAM permissions for Secrets Manager
   - Ensure secret contains `host` or `endpoint` field

3. **Session Manager connection fails**
   - Install AWS Session Manager plugin
   - Verify EC2 instance has SSM agent installed and running
   - Check IAM roles allow SSM access

4. **Port forwarding "no such host" error**
   - Usually indicates RDS endpoint parsing issue
   - Try specifying RDS endpoint manually
   - Check VPC connectivity between EC2 and RDS

**Debug Commands:**
```bash
# Check AWS configuration
aws configure list

# Test AWS connectivity
aws sts get-caller-identity

# Verify Session Manager plugin
session-manager-plugin

# Check EC2 SSM connectivity
aws ssm describe-instance-information
```