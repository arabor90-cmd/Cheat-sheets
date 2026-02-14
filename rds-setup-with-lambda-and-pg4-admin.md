# AWS RDS PostgreSQL Setup Guide

This guide covers how to create a PostgreSQL database on AWS RDS, configure security rules, and connect to it from your local machine (pgAdmin 4) and AWS Lambda.

---

## 1. Create the RDS Database

1.  **Go to RDS Console**: Navigate to [AWS RDS](https://console.aws.amazon.com/rds/).
2.  **Create Database**: Click **Create database**.
3.  **Engine Options**:
    *   **Engine Type**: PostgreSQL
    *   **Version**: Latest (e.g., PostgreSQL 15 or 16)
4.  **Templates**: Choose **Free Tier** (if eligible) or **Dev/Test**.
5.  **Settings**:
    *   **DB instance identifier**: `debugflow-db-instance`
    *   **Master username**: `postgres` (or your choice)
    *   **Master password**: *[Set a strong password]*
6.  **Instance Configuration**:
    *   **DB instance class**: `db.t3.micro` or `db.t4g.micro` (Free Tier eligible).
7.  **Connectivity**:
    *   **VPC**: Select your default VPC or a custom one.
    *   **Public Access**: **Yes** (Highly recommended for easiest setup with Lambda and local debugging).
    *   **Security Group**: Choose **Create new** and name it `debugflow-rds-sg`.
8.  **Additional Configuration**:
    *   **Initial database name**: `debugflow_db` (Must match your `DATABASE_URL`).
9.  **Create**: Click **Create database**. This takes about 5–10 minutes.

---

## 2. Configure Security Group (Inbound Rules)

Your database acts like a locked server. You must "open" the door for pgAdmin and Lambda.

1.  In the RDS Console, click your new database name.
2.  Under **Connectivity & security**, click the link under **VPC security groups**.
3.  Select the security group → **Inbound rules** → **Edit inbound rules**.

Add the following two rules:

| Type | Protocol | Port | Source | Description |
| :--- | :--- | :--- | :--- | :--- |
| **PostgreSQL** | TCP | `5432` | **My IP** | Allows your computer to connect via pgAdmin |
| **PostgreSQL** | TCP | `5432` | `0.0.0.0/0` | Allows Lambda (outside VPC) to connect. *Ensure password is strong.* |

> [!TIP]
> **If Lambda is in a VPC**: Instead of `0.0.0.0/0`, set the Source to the **Security Group ID** of your Lambda function. This is more secure.

---

## 3. Connect from pgAdmin 4

1.  **Find Endpoint**: In RDS Console, copy the **Endpoint** (e.g., `xxx.us-east-1.rds.amazonaws.com`).
2.  **Open pgAdmin**: Right-click **Servers** → **Register** → **Server**.
3.  **General Tab**: Name = `DebugFlow RDS`.
4.  **Connection Tab**:
    *   **Host name/address**: *[Paste your Endpoint]*
    *   **Port**: `5432`
    *   **Maintenance database**: `debugflow_db`
    *   **Username**: `postgres`
    *   **Password**: *[Your Master Password]*
5.  **SSL Tab**: Set **SSL Mode** = `Require`.
6.  **Save**: You should now see your database and tables.

---

## 4. Connect from AWS Lambda

### Step 1: Set Environment Variable
In the Lambda Console → **Configuration** → **Environment variables**, add:

*   **Key**: `DATABASE_URL`
*   **Value**: `postgresql://postgres:YOUR_PASSWORD@YOUR_ENDPOINT:5432/debugflow_db`

### Step 2: Code Implementation
Our project is already configured to handle RDS connections. The key requirements are met in:
*   `src/db/postgres_connect.js`: Handles SSL connections using `rejectUnauthorized: false`.
*   `lambda.js`: Sets `context.callbackWaitsForEmptyEventLoop = false` to prevent timeouts.

### Step 3: SSL Certificate (Automatic)
The project includes `certs/global-bundle.pem`. AWS RDS instances require this for encrypted connections. Our code is programmed to look for this file automatically when running in production.

---

## Troubleshooting Connectivity

*   **Timed Out?**: 99% of the time, this is a **Security Group** issue. Check that your "My IP" rule is correct in the RDS Security Group.
*   **Authentication Failed?**: Double-check your username and password in the `DATABASE_URL`.
*   **Lambda can't reach RDS?**: If RDS has **Public Access: No**, your Lambda **must** be in the same VPC. If RDS has **Public Access: Yes**, Lambda can connect normally without VPC settings.
*   **SSL Errors?**: Ensure `rejectUnauthorized: false` is set in your database configuration (our project does this by default).
