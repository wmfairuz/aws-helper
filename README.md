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
- Appropriate AWS IAM permissions for EC2, RDS, SSM, and Secrets Manager

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
| `aws-tail` | Tail CloudWatch log streams |

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

# Switch to a named profile
aws-profile prod
```

---

### `aws-ec2-list`

Display all EC2 instances in a formatted table showing instance IDs, names, and states.

**Usage:**
```bash
aws-ec2-list
```

**Sample Output:**
```
---------------------------------------------------------------------------
|                           DescribeInstances                             |
+----------------------+------------------+----------+
|  i-0123456789abcdef0 |  my-app-server   | running  |
|  i-0abcdef123456789  |  web-server      | stopped  |
+----------------------+------------------+----------+
```

---

### `aws-asg-refresh`

Start an instance refresh for an Auto Scaling Group.

**Usage:**
```bash
aws-asg-refresh
```

**Interactive Flow:**
1. Lists all Auto Scaling Groups with capacity settings
2. Prompts for ASG selection
3. Shows downtime warning (`MinHealthyPercentage=0`)
4. Asks for confirmation before proceeding

**Warning:** This uses aggressive settings (`MinHealthyPercentage=0`, `InstanceWarmup=0`) that **will cause downtime**. Use only during maintenance windows.

---

### `aws-rds-info`

Retrieve RDS database connection credentials from AWS Secrets Manager.

**Usage:**
```bash
aws-rds-info
```

**Prerequisites:**
- `AWS_RDS_SECRET_ID` environment variable must be set

**Example:**
```bash
export AWS_RDS_SECRET_ID="myapp/database/password"
aws-rds-info
```

**Sample Output:**
```json
{
  "username": "admin",
  "password": "secret123",
  "engine": "mysql",
  "host": "my-rds.cluster-xyz.ap-southeast-5.rds.amazonaws.com",
  "port": 3306,
  "dbname": "production"
}
```

---

### `aws-ssh`

Connect to an EC2 instance via SSH using AWS Session Manager (no public IP or SSH key required).

**Usage:**
```bash
aws-ssh
```

**Interactive Flow:**
1. Lists all running EC2 instances
2. Prompts for instance selection
3. Establishes SSH session via Session Manager

**Sample Output:**
```
AWS Profile: my-profile

Fetching running EC2 instances...
Available running EC2 instances:
=====================================
1) i-0123456789abcdef0 - my-app-server - Private: 10.0.10.46 - ASG: my-app-asg
2) i-0abcdef123456789  - web-server    - Private: 10.0.10.52 - (not in ASG)

Select instance number (1-2): 1
Connecting to instance: i-0123456789abcdef0
Starting SSH session via AWS Session Manager...
```

---

### `aws-scp`

Transfer files to/from EC2 instances using SSH over Session Manager (recommended).

**Usage:**
```bash
aws-scp <source> <destination> [username]
```

**Parameters:**
- `source`: Source path (relative/absolute local, or absolute remote)
- `destination`: Destination path (relative/absolute local, or absolute remote)
- `username`: SSH username (optional, defaults to `ubuntu`)

**Direction detection:**
- Local → remote: local path is relative, remote path is absolute (`/`)
- Remote → local: remote path is absolute (`/`), local path is relative

**Examples:**
```bash
aws-scp ./config.json /opt/app/config/       # Upload
aws-scp /var/log/app.log ./logs/             # Download
aws-scp ./file.txt /home/ec2-user/ ec2-user  # Custom username
```

**Prerequisites:**
- SSH public key added to the instance's `~/.ssh/authorized_keys`

---

### `aws-scp-ssm`

Transfer files to/from EC2 instances using SSM commands. Fallback when SSH keys are unavailable.

**Usage:**
```bash
aws-scp-ssm <source> <destination>
```

**Notes:**
- Slower than `aws-scp` — uses base64 encoding through SSM
- No SSH keys required
- May struggle with very large files due to SSM command output limits

---

### `aws-db-fwdport`

Create a port forwarding tunnel to an RDS database through an EC2 instance via AWS Session Manager.

**Usage:**
```bash
aws-db-fwdport [local-port] [rds-endpoint]
```

**Parameters:**
- `local-port` (optional): Local port to bind (default: `33060`)
- `rds-endpoint` (optional): RDS hostname. If omitted, you'll be prompted to select from available RDS instances.

**Interactive Flow:**
1. If no endpoint given — lists available RDS instances and prompts for selection
2. Lists running EC2 instances and prompts for selection (used as the SSM tunnel proxy)
3. Starts port forwarding session

**Examples:**
```bash
# Interactive: pick RDS instance and EC2 proxy, use default local port 33060
aws-db-fwdport

# Interactive RDS selection, custom local port
aws-db-fwdport 3307

# Explicit RDS endpoint, default local port
aws-db-fwdport "" my-rds.cluster-xyz.ap-southeast-5.rds.amazonaws.com

# Fully explicit
aws-db-fwdport 3307 my-rds.cluster-xyz.ap-southeast-5.rds.amazonaws.com
```

**Sample Output:**
```
AWS Profile: my-profile

Fetching RDS instances...
Available RDS instances:
========================
1) my-app-db   - my-app-db.cluster-xyz.ap-southeast-5.rds.amazonaws.com (available, db.t3.medium)
2) my-app-db-2 - my-app-db-2.cluster-abc.ap-southeast-5.rds.amazonaws.com (available, db.t3.medium)

Select RDS instance number (1-2): 1

Fetching running EC2 instances...
Available running EC2 instances:
=====================================
1) i-0123456789abcdef0 - my-app-server - Private: 10.0.10.46 - ASG: my-app-asg

Select instance number (1-1): 1
Starting port forwarding session...
Instance: i-0123456789abcdef0
RDS Host: my-app-db.cluster-xyz.ap-southeast-5.rds.amazonaws.com
Remote Port: 3306 -> Local Port: 33060

Starting session with SessionId: botocore-session-123456789
Port 33060 opened for sessionId botocore-session-123456789.
Waiting for connections...
```

**Connect after tunnel is up:**
```bash
mysql -h 127.0.0.1 -P 33060 -u admin -p
```

**Notes:**
- Remote port is always 3306 (MySQL/MariaDB)
- Session stays active until you press `Ctrl+C`

---

### `aws-tail`

Tail CloudWatch log streams interactively.

**Usage:**
```bash
aws-tail [-f] [-n NUM] [--since TIME]
```

**Options:**
- `-f`: Follow mode — stream new log events in real time
- `-n NUM`: Limit output to last N lines (static mode only)
- `--since TIME`: How far back to look (default: `1h`). Examples: `30m`, `2h`, `1d`

**Interactive Flow:**
1. Lists available CloudWatch log groups
2. Prompts for selection
3. Streams or displays logs

**Examples:**
```bash
aws-tail                        # Last 1h of logs from selected group
aws-tail -f                     # Follow logs in real time
aws-tail -n 50                  # Last 50 lines
aws-tail -f --since 30m         # Follow from 30 minutes ago
```

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `AWS_DEFAULT_REGION` | Set to `ap-southeast-5` (auto-exported) |
| `AWS_REGION` | Set to `ap-southeast-5` (auto-exported) |
| `AWS_PROFILE` | Current profile, managed by `aws-profile` |
| `AWS_RDS_SECRET_ID` | Secrets Manager path for RDS credentials (used by `aws-rds-info`) |

---

## Troubleshooting

**"No running EC2 instances found"**
- Verify the correct AWS profile/region is active
- Check IAM permissions for `ec2:DescribeInstances`

**"No RDS instances found"**
- Check IAM permissions for `rds:DescribeDBInstances`
- Verify instances exist in the current region/profile

**Session Manager connection fails**
- Install the [AWS Session Manager plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
- Ensure the EC2 instance has the SSM agent running
- Check the instance's IAM role allows SSM access

**Port forwarding "no such host" error**
- Verify VPC connectivity between the EC2 instance and RDS
- Try specifying the RDS endpoint manually

**Debug commands:**
```bash
aws configure list              # Check current config
aws sts get-caller-identity     # Test credentials
session-manager-plugin          # Verify SSM plugin installed
aws ssm describe-instance-information  # Check SSM-connected instances
```
