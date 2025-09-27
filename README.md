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
| `aws-rds-info` | Retrieve RDS credentials from Secrets Manager |
| `aws-ssh` | SSH into EC2 instance via Session Manager |
| `aws-db-fwdport` | Port forward to RDS through EC2 bastion |

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

**Notes:**
- Fetches from secret ID: `myapp/database/password`
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

### `aws-db-fwdport`

Create a port forwarding tunnel to RDS database through an EC2 bastion host.

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

**Interactive Flow:**
1. Fetches RDS endpoint from Secrets Manager (if not provided)
2. Lists running EC2 instances for bastion selection
3. Establishes port forwarding tunnel
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
- RDS endpoint auto-fetched from `myapp/database/password` secret
- Automatically strips port from RDS endpoint if present
- Session remains active until manually terminated (Ctrl+C)

---

## Environment Variables

The following environment variables are automatically set:

- `AWS_DEFAULT_REGION`: ap-southeast-5
- `AWS_REGION`: ap-southeast-5
- `AWS_PROFILE`: Managed by `aws-profile` function

## Troubleshooting

**Common Issues:**

1. **"No running EC2 instances found"**
   - Check AWS profile and region settings
   - Verify EC2 instances are in running state
   - Ensure proper IAM permissions for EC2 DescribeInstances

2. **"Could not fetch RDS endpoint from Secrets Manager"**
   - Verify secret exists: `myapp/database/password`
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