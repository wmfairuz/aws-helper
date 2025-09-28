# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an AWS helper utility repository containing bash functions and configurations for managing AWS resources. The main component is `.aws-config`, which provides a collection of bash functions for common AWS operations.

## Architecture

The repository is minimal and focused:

- `.aws-config`: Main configuration file containing AWS helper functions and environment setup
- Functions are designed to be sourced into a bash environment for interactive use

## Core Functions

The `.aws-config` file provides these key functions:

- `aws-profile()`: Switch between AWS profiles or list available profiles
- `aws-rds-info()`: Retrieve RDS database credentials from AWS Secrets Manager
- `aws-ec2-list()`: List EC2 instances with names and states in table format
- `aws-db-fwdport()`: Start port forwarding session to RDS through EC2 instance via SSM

## Usage

To use these functions, source the configuration file:
```bash
source .aws-config
```

The functions expect:
- AWS CLI to be installed and configured
- Proper AWS credentials and profiles set up
- `jq` installed for JSON parsing
- Session Manager plugin for port forwarding functionality

## Environment Variables

- `AWS_DEFAULT_REGION` and `AWS_REGION`: Set to `ap-southeast-5`
- `AWS_PROFILE`: Managed by `aws-profile()` function

## Notes

- The `aws-db-fwdport()` function contains placeholder values that need to be customized:
  - Replace `i-YOUR-EC2-INSTANCE-ID` with actual EC2 instance ID
  - Replace `your-rds-endpoint` with actual RDS endpoint
- Port forwarding maps remote port 3306 to local port 33060