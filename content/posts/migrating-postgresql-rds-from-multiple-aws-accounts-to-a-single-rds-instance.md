+++
date = '2025-08-28T21:14:07+02:00'
draft = false
title = 'Migrating Postgresql Rds From Multiple Aws Accounts to a Single Rds Instance'
+++
# Table of Contents
- [The Scenario](#the-scenario)
- [Our Architecture](#our-architecture)
- [Prerequisites](#prerequisites)
- [The Migration Strategy: Why `pg_dump` and `pg_restore`?](#the-migration-strategy-why-pg_dump-and-pg_restore)
- [Step-by-Step Migration Guide](#step-by-step-migration-guide)
  - [Phase 1: Preparing the Target (Non-Prod) Account](#phase-1-preparing-the-target-non-prod-account)
  - [Phase 2: Preparing the Source Accounts (A & B)](#phase-2-preparing-the-source-accounts-a--b)
  - [Phase 3: Executing the Migration Script](#phase-3-executing-the-migration-script)
  - [Phase 4: Post-Migration Validation](#phase-4-post-migration-validation)
- [Final Considerations](#final-considerations)

As organizations grow, it's common for development and testing environments to become siloed in separate AWS accounts. While this is great for security and billing isolation, it can lead to increased management overhead and costs. A common strategy to streamline operations is to consolidate these non-production databases into a single, shared RDS instance in a central "Non-Prod" account.

In this guide, we'll walk through a robust, script-based method for migrating multiple PostgreSQL RDS instances from different source AWS accounts into a single target RDS instance. We'll use a dedicated EC2 "migration host" and standard PostgreSQL tools (`pg_dump` and `pg_restore`) to ensure a reliable and repeatable process.

### The Scenario

Imagine you have two source AWS accounts and one target account:
*   **Source Account A**: Contains an RDS instance with a database for one application (`app_db_a`).
*   **Source Account B**: Contains an RDS instance with a database for another application or environment (`app_db_b`).
*   **Target Non-Prod Account**: Where we want to host a single, larger RDS instance containing both `app_db_a` and `app_db_b` as separate logical databases.

### Our Architecture

We will perform the migration from a central EC2 instance in the target account. This instance will assume IAM roles in the source accounts to gain the necessary access to pull database backups.



## Prerequisites

Before you begin, ensure you have the following in place:

1.  **IAM Permissions**: You need administrative access in all three AWS accounts to manage IAM Roles, Policies, EC2, RDS, and VPC settings.
2.  **Networking**:
    *   **VPC Peering**: You must establish and accept a VPC peering connection between the Target VPC and each Source VPC.
    *   **Route Tables**: Update the route tables in all VPCs to direct traffic correctly across the peering connections.
    *   **Security Groups**: This is critical.
        *   **Source RDS Security Groups**: Must allow inbound traffic on PostgreSQL port `5432` from the **private IP address** of the EC2 migration host.
        *   **Target RDS Security Group**: Must allow inbound traffic on port `5432` from the EC2 migration host.
3.  **Credentials Management**: You have the master credentials for all RDS instances. **It is a security best practice to store these credentials in AWS Secrets Manager** in their respective accounts, and this guide will assume you are doing so.
4.  **Tools**: A Linux-based EC2 instance with the **AWS CLI** and the **PostgreSQL client tools** installed.

## The Migration Strategy: Why `pg_dump` and `pg_restore`?

For this task, we're choosing a logical backup-and-restore method.

*   **`pg_dump`**: Creates a consistent, portable text or binary-format backup of a single database.
*   **`pg_restore`**: Restores a database from an archive created by `pg_dump`.

This approach is ideal because it's:
*   **Flexible**: It easily handles migrations between different PostgreSQL versions.
*   **Granular**: It allows us to dump a database from one server and restore it as a different logical database on another, which is exactly what we need.
*   **Reliable**: It's the standard, battle-tested method for PostgreSQL migrations.

**Downtime Note**: This method requires a brief downtime for the source applications to ensure the database dump is a consistent snapshot. Plan your migration window accordingly.

---

## Step-by-Step Migration Guide

### Phase 1: Preparing the Target (Non-Prod) Account

1.  **Launch the EC2 Migration Host**
    *   Launch an Amazon Linux 2 EC2 instance (e.g., `t3.medium`) in the same VPC as your target RDS instance.
    *   Attach an IAM Role to it, which we'll create in the next step.

2.  **Create an IAM Role for the EC2 Host (`MigrationHostRole`)**
    This role will allow our EC2 instance to assume roles in the source accounts. Create a policy named `AssumeCrossAccountRolePolicy` and attach it to the role.

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "sts:AssumeRole",
                "Resource": [
                    "arn:aws:iam::SOURCE_A_ACCOUNT_ID:role/RdsCrossAccountAccessRole",
                    "arn:aws:iam::SOURCE_B_ACCOUNT_ID:role/RdsCrossAccountAccessRole"
                ]
            }
        ]
    }
    ```

3.  **Prepare the Target RDS Instance**
    *   Provision your consolidated PostgreSQL RDS instance if it doesn't already exist.
    *   Connect to the instance with a client like `psql` and create the empty databases that will receive the migrated data.
    ```sql
    -- Connect as the master user
    psql -h <your-target-rds-endpoint> -U <master_user> -d postgres

    -- Run these commands
    CREATE DATABASE app_db_a;
    CREATE DATABASE app_db_b;
    ```

### Phase 2: Preparing the Source Accounts (A & B)

***Perform these steps in each source account.***

1.  **Create the Cross-Account IAM Role (`RdsCrossAccountAccessRole`)**
    This is the role that our EC2 host will assume.
    *   **Role Name**: Use a consistent name, e.g., `RdsCrossAccountAccessRole`.
    *   **Trust Relationship**: Configure the role to trust the `MigrationHostRole` from your Target Account.

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::TARGET_ACCOUNT_ID:role/MigrationHostRole"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    }
    ```

2.  **Attach a Permissions Policy**
    Create and attach a policy to this role that grants permissions to read RDS details and fetch the database password from Secrets Manager.

    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "rds:DescribeDBInstances",
                "Resource": "*"
            },
            {
                "Effect": "Allow",
                "Action": "secretsmanager:GetSecretValue",
                "Resource": "arn:aws:secretsmanager:REGION:ACCOUNT_ID:secret:YOUR_RDS_SECRET_NAME-*"
            }
        ]
    }
    ```

### Phase 3: Executing the Migration Script

Now for the main event! Connect to your EC2 migration host and follow these steps.

1.  **Install Tools**
    ```bash
    sudo yum update -y
    sudo yum install -y postgresql15 jq  # Use the client version matching your DB
    ```

2.  **The Automation Script**
    Create a file named `migrate_dbs.sh` and paste the script below. This script automates assuming roles, fetching credentials, dumping the source database, and restoring it to the target.

    **Before you run it, carefully update all the placeholder variables in the configuration section.**

    ```bash
    #!/bin/bash

    # Exit immediately if a command exits with a non-zero status.
    set -e

    # --- Configuration: UPDATE THESE VARIABLES ---
    # Target Account Details
    TARGET_RDS_IDENTIFIER="your-target-rds-instance-id"
    TARGET_SECRET_ARN="arn:aws:secretsmanager:us-east-1:TARGET_ACCOUNT_ID:secret/your-target-secret"
    TARGET_REGION="us-east-1"

    # Source A Details
    SOURCE_A_ACCOUNT_ID="111111111111"
    SOURCE_A_ROLE_ARN="arn:aws:iam::${SOURCE_A_ACCOUNT_ID}:role/RdsCrossAccountAccessRole"
    SOURCE_A_RDS_IDENTIFIER="source-a-rds-instance-id"
    SOURCE_A_SECRET_ARN="arn:aws:secretsmanager:us-east-1:${SOURCE_A_ACCOUNT_ID}:secret/your-source-a-secret"
    SOURCE_A_DB_NAME="source_database_a" # The actual DB name in the source RDS
    TARGET_A_DB_NAME="app_db_a"          # The new DB name in the target RDS
    SOURCE_A_REGION="us-east-1"

    # Source B Details
    SOURCE_B_ACCOUNT_ID="222222222222"
    SOURCE_B_ROLE_ARN="arn:aws:iam::${SOURCE_B_ACCOUNT_ID}:role/RdsCrossAccountAccessRole"
    SOURCE_B_RDS_IDENTIFIER="source-b-rds-instance-id"
    SOURCE_B_SECRET_ARN="arn:aws:secretsmanager:us-east-1:${SOURCE_B_ACCOUNT_ID}:secret/your-source-b-secret"
    SOURCE_B_DB_NAME="source_database_b" # The actual DB name in the source RDS
    TARGET_B_DB_NAME="app_db_b"          # The new DB name in the target RDS
    SOURCE_B_REGION="us-east-1"
    # --- End Configuration ---

    # --- Reusable Migration Function ---
    migrate_source() {
        local NICKNAME=$1; local ROLE_ARN=$2; local RDS_IDENTIFIER=$3;
        local SECRET_ARN=$4; local SOURCE_DB=$5; local TARGET_DB=$6;
        local REGION=$7; local DUMP_FILE="/tmp/${TARGET_DB}.dump"

        echo "--- Starting Migration for: ${NICKNAME} ---"

        # 1. Assume Role in the source account
        echo "--> Assuming role in source account..."
        CREDS=$(aws sts assume-role --role-arn "${ROLE_ARN}" --role-session-name "RDSCrossAccountMigration" --output json)
        export AWS_ACCESS_KEY_ID=$(echo "${CREDS}" | jq -r .Credentials.AccessKeyId)
        export AWS_SECRET_ACCESS_KEY=$(echo "${CREDS}" | jq -r .Credentials.SecretAccessKey)
        export AWS_SESSION_TOKEN=$(echo "${CREDS}" | jq -r .Credentials.SessionToken)

        # 2. Get Source RDS Connection Details
        echo "--> Fetching source RDS details for ${RDS_IDENTIFIER}..."
        SOURCE_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${RDS_IDENTIFIER}" --region "${REGION}" | jq -r .DBInstances[0].Endpoint.Address)
        SOURCE_SECRET=$(aws secretsmanager get-secret-value --secret-id "${SECRET_ARN}" --region "${REGION}" | jq -r .SecretString)
        SOURCE_USER=$(echo "${SOURCE_SECRET}" | jq -r .username)
        export PGPASSWORD=$(echo "${SOURCE_SECRET}" | jq -r .password)
        if [ -z "$SOURCE_ENDPOINT" ]; then echo "Error: Failed to get source endpoint."; exit 1; fi

        # 3. Dump the source database
        echo "--> Dumping database '${SOURCE_DB}'..."
        pg_dump -Fc -v -h "${SOURCE_ENDPOINT}" -U "${SOURCE_USER}" -d "${SOURCE_DB}" -f "${DUMP_FILE}"
        echo "--> Dump complete: ${DUMP_FILE}"

        # Unset temporary AWS credentials
        unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

        # 4. Get Target RDS Connection Details
        echo "--> Fetching target RDS details..."
        TARGET_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${TARGET_RDS_IDENTIFIER}" --region "${TARGET_REGION}" | jq -r .DBInstances[0].Endpoint.Address)
        TARGET_SECRET=$(aws secretsmanager get-secret-value --secret-id "${TARGET_SECRET_ARN}" --region "${TARGET_REGION}" | jq -r .SecretString)
        TARGET_USER=$(echo "${TARGET_SECRET}" | jq -r .username)
        export PGPASSWORD=$(echo "${TARGET_SECRET}" | jq -r .password)
        if [ -z "$TARGET_ENDPOINT" ]; then echo "Error: Failed to get target endpoint."; exit 1; fi

        # 5. Restore to the target database
        echo "--> Restoring dump to database '${TARGET_DB}'..."
        pg_restore -v -h "${TARGET_ENDPOINT}" -U "${TARGET_USER}" -d "${TARGET_DB}" "${DUMP_FILE}"
        echo "--> Restore complete."

        # 6. Cleanup
        rm "${DUMP_FILE}"
        unset PGPASSWORD
        echo "--- Migration for ${NICKNAME} finished successfully! ---"
    }

    # --- Main Execution ---
    migrate_source "Source A" "$SOURCE_A_ROLE_ARN" "$SOURCE_A_RDS_IDENTIFIER" "$SOURCE_A_SECRET_ARN" "$SOURCE_A_DB_NAME" "$TARGET_A_DB_NAME" "$SOURCE_A_REGION"
    migrate_source "Source B" "$SOURCE_B_ROLE_ARN" "$SOURCE_B_RDS_IDENTIFIER" "$SOURCE_B_SECRET_ARN" "$SOURCE_B_DB_NAME" "$TARGET_B_DB_NAME" "$SOURCE_B_REGION"

    echo "ALL MIGRATIONS COMPLETED."
    ```
3.  **Run the Script**
    ```bash
    chmod +x migrate_dbs.sh
    ./migrate_dbs.sh
    ```

### Phase 4: Post-Migration Validation

Always verify your work.
1.  **Connect to the Target RDS**: Use `psql` to connect to your new consolidated instance.
2.  **Check Databases**: List all databases to ensure `app_db_a` and `app_db_b` are present (`\l`).
3.  **Inspect Data**: Connect to each new database (`\c app_db_a`) and inspect the tables (`\dt`). Run a few `SELECT count(*)` queries on important tables to ensure data has been transferred correctly.
4.  **Check Ownership and Permissions**: `pg_dump` preserves ownership information. If the original user roles don't exist on the target RDS, the objects will be owned by the master user. You may need to create the necessary roles and `GRANT` permissions manually after the migration.

## Final Considerations

*   **Rollback Plan**: Your safest rollback path is the original source RDS instances. Do not decommission them until the new consolidated instance has been fully tested and validated by your application teams.
*   **Large Databases**: For extremely large databases (hundreds of gigabytes or more), this `pg_dump` method can be slow and require significant disk space on the migration host. For those cases, consider using the AWS Database Migration Service (DMS).
*   **Cleanup**: Once the migration is complete and verified, remember to terminate the EC2 migration host to save costs and remove the temporary IAM Roles and Policies if they are no longer needed.

Congratulations! You have successfully consolidated your RDS instances, simplifying your non-production environment and potentially saving on operational costs.