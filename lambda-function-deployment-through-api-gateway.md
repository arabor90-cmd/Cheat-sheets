# DebugFlow Backend — AWS Lambda Deployment Guide (API Gateway)

This guide covers deploying the DebugFlow Express backend to **AWS Lambda** using **API Gateway** with **Buffered Response** mode, and connecting your **RDS PostgreSQL database** via **pgAdmin 4**.
---
## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Lambda Handler (`lambda.js`)](#lambda-handler-lambdajs)
- [Step 1: Create the Lambda Function](#step-1-create-the-lambda-function)
- [Step 2: Configure Environment Variables](#step-2-configure-environment-variables)
- [Step 3: Create an API Gateway](#step-3-create-an-api-gateway)
- [Step 4: Configure CORS](#step-4-configure-cors)
- [Step 5: Build & Deploy](#step-5-build--deploy)
- [Step 6: Verify Deployment](#step-6-verify-deployment)
- [Connecting to RDS from pgAdmin 4](#connecting-to-rds-from-pgadmin-4)
- [Updating Your Frontend](#updating-your-frontend)
- [Troubleshooting](#troubleshooting)
---
## Architecture Overview
```
┌──────────┐     HTTPS      ┌───────────────┐              ┌──────────┐     ┌───────────┐
│ Frontend │ ──────────────▶ │ API Gateway   │ ──────────▶  │  Lambda  │ ──▶ │  Express  │
│ (Vercel) │ ◀────────────── │ (HTTP API)    │ ◀──────────  │ Handler  │ ◀── │   App     │
└──────────┘                 └───────────────┘              └──────────┘     └───────────┘
                                                                                   │
                                                    ┌──────────────────────────────┼──────────────────┐
                                                    ▼                              ▼                  ▼
                                               PostgreSQL                    Firebase Auth      HuggingFace AI
                                                (AWS RDS)                   (Token Verify)     (Chat/Embeddings)
```
**How it works:**
- **API Gateway (HTTP API)** sits in front of Lambda as a managed HTTPS endpoint with routing, throttling, and CORS support
- **`serverless-http`** wraps your Express app so it runs inside Lambda
- **Buffered mode**: Lambda executes the full request, and API Gateway returns the complete response to the client
- Express handles all middleware (**authentication**, **rate limiting**, **error handling**) internally
**Why API Gateway over Function URL?**
- Custom domain name support
- Built-in throttling and usage plans
- Request/response validation
- Authorization options (IAM, JWT, Lambda authorizers)
- CloudWatch metrics and access logging out of the box
---
## Prerequisites
- **AWS Account** with permissions to create Lambda functions, API Gateway, IAM roles, and VPC resources
- **Node.js 20+** installed locally
- **AWS CLI** (optional, for CLI-based deployment)
- **pgAdmin 4** installed ([download here](https://www.pgadmin.org/download/))
- A **PostgreSQL database** on AWS RDS
- A **Firebase project** with Authentication enabled
- **HuggingFace API key** (for AI chat and embeddings)
- **Pinecone API key** (for vector database / RAG)
---
## Lambda Handler (`lambda.js`)
This is the entry point for your Lambda function. It wraps the Express app with `serverless-http` for buffered response mode.
Create or verify this file exists at `backend/lambda.js`:
```javascript
import serverless from 'serverless-http';
import { app } from './src/app.js';
// Wrap the Express app with serverless-http (buffered mode)
const serverlessHandler = serverless(app);
// Single handler for ALL routes
export const handler = async (event, context) => {
    context.callbackWaitsForEmptyEventLoop = false;
    return serverlessHandler(event, context);
};
```
**Key details:**
| Line | Purpose |
|------|---------|
| `serverless(app)` | Converts Lambda events into Express `req`/`res` objects automatically |
| `callbackWaitsForEmptyEventLoop = false` | Prevents Lambda from waiting for idle DB connections before returning |
| `export const handler` | The function Lambda invokes — must match the **Handler** setting in Runtime Settings |
> **Note**: This handler supports ALL Express routes (chat, sessions, profile, admin, etc.) through a single Lambda function.
---
## Step 1: Create the Lambda Function
### Via AWS Console
1. Go to **AWS Lambda** → **Create function**
2. Choose **Author from scratch**
3. Configure:
   - **Function name**: `debugflow-backend` (or your preferred name)
   - **Runtime**: `Node.js 20.x` or higher (Node.js 24.x recommended)
   - **Architecture**: `x86_64`
   - **Handler**: `lambda.handler`
4. Under **Advanced settings**:
   - Check **Enable VPC** if your RDS database is in a VPC
   - Select the appropriate **VPC**, **Subnets** (private subnets), and **Security Groups**
5. Click **Create function**
### Post-Creation Settings
Go to **Configuration** → **General configuration** → **Edit**:
| Setting | Recommended Value | Why |
|---------|-------------------|-----|
| **Memory** | `512 MB` | More memory = more CPU. AI responses need processing power |
| **Timeout** | `60 seconds` | AI chat responses can take 10-30s to generate |
| **Ephemeral storage** | `512 MB` (default) | Sufficient for the deployment package |
### Execution Role Permissions
Your Lambda's execution role needs these managed policies:
| Policy | Purpose |
|--------|---------|
| `AWSLambdaBasicExecutionRole` | CloudWatch Logs |
| `AWSLambdaVPCAccessExecutionRole` | VPC access (if using VPC) |
> **⚠️ VPC Requirement**: If your Lambda is in a VPC (required for private RDS), you **must** have a **NAT Gateway** in a public subnet for outbound internet access. Without it, Firebase auth, HuggingFace API, and Pinecone calls will time out.
---
## Step 2: Configure Environment Variables
Go to **Configuration** → **Environment variables** → **Edit** and add:
### Required Variables
| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@your-rds-endpoint:5432/debugflow_db` |
| `HUGGINGFACE_API_KEY` | HuggingFace Inference API key | `hf_xxxxxxxxxxxxxxxx` |
| `PINECONE_API_KEY` | Pinecone vector DB API key | `pcsk_xxxxxxxxxxxxxxxx` |
### Firebase (Required for Auth)
Your Firebase Admin SDK service account credentials must be accessible. This is typically configured in `src/lib/firebaseAdmin.js` using a service account JSON file included in your deployment package or via environment variables.
### Optional Variables
| Variable | Description | Default |
|----------|-------------|---------|
| `HUGGINGFACE_CHAT_MODEL` | Chat model name | `deepseek-ai/DeepSeek-V3.1:novita` |
| `HUGGINGFACE_CHAT_TIMEOUT_MS` | Chat timeout (ms) | `8000` |
| `HUGGINGFACE_EMBEDDING_MODEL` | Embedding model | `sentence-transformers/all-MiniLM-L6-v2` |
| `PINECONE_INDEX_NAME` | Pinecone index name | `debugflow-logs` |
| `SUPER_USER_ID` | Firebase UID for initial super user | — |
| `SUPER_USER_EMAIL` | Email for initial super user | — |
| `EMERGENCY_OVERRIDE_ID` | Firebase UID for emergency admin recovery | — |
| `EMERGENCY_OVERRIDE_EMAIL` | Email for emergency admin recovery | — |
| `LOG_LEVEL` | Logging level | `info` |
---
## Step 3: Create an API Gateway
### Create an HTTP API
1. Go to **API Gateway** → **Create API**
2. Choose **HTTP API** → **Build**
3. Click **Add integration**:
   - **Integration type**: Lambda
   - **Lambda function**: Select `debugflow-backend`
   - **API name**: `debugflow-api`
4. Click **Next**
### Configure Routes
5. On the **Configure routes** page, set up a catch-all route:
   - **Method**: `ANY`
   - **Resource path**: `/{proxy+}`
   - **Integration target**: `debugflow-backend`
6. Click **Next**
This catch-all route forwards ALL requests to your Express app, which handles its own routing internally.
### Configure Stage
7. On the **Configure stages** page:
   - **Stage name**: `$default` (auto-deploy enabled)
8. Click **Next** → **Create**
### Get Your API URL
After creation, you'll see your **Invoke URL**:
```
https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com
```
This is your production backend URL.
### Configure Timeout (Important)
By default, API Gateway has a **30-second timeout**. For AI chat responses that may take longer:
1. Go to your API → **Integrations** → Select the Lambda integration
2. Click **Manage integration** → **Edit**
3. Set **Timeout**: `30000` ms (maximum for HTTP API is 30s)
> **Note**: If your AI responses regularly exceed 30 seconds, consider using a **Function URL** instead of API Gateway (no timeout limit beyond Lambda's own 15-minute max).
---
## Step 4: Configure CORS
### Option A: Let Express Handle CORS (Recommended)
Your Express `app.js` already has CORS configured. For API Gateway to pass requests through without interference:
1. Go to your API → **CORS**
2. **Do NOT configure CORS** in API Gateway
3. Let API Gateway forward the raw request to Lambda, and Express responds with the correct CORS headers
This avoids duplicate header conflicts.
### Option B: Configure CORS in API Gateway
If you prefer API Gateway to handle CORS preflight (OPTIONS) requests:
1. Go to your API → **CORS** → **Configure**
2. Set:
   - **Access-Control-Allow-Origin**: `https://your-app.vercel.app`
   - **Access-Control-Allow-Methods**: `GET, POST, PUT, DELETE, PATCH, OPTIONS`
   - **Access-Control-Allow-Headers**: `Content-Type, Authorization, X-Requested-With`
   - **Access-Control-Allow-Credentials**: `true`
3. **If using this option**: Disable Express CORS in `app.js` to prevent duplicate headers
> **⚠️ Warning**: Never enable CORS in both API Gateway AND Express simultaneously. This creates duplicate `Access-Control-Allow-Origin` headers, which browsers reject.
---
## Step 5: Build & Deploy
### Using the Deploy Script
From the `backend/` directory:
```bash
# 1. Make the script executable (first time only)
chmod +x zip_deploy.sh
# 2. Build the deployment package
./zip_deploy.sh
```
**What the script does:**
1. Creates a clean `dist/` build directory
2. Copies `lambda.js`, `src/`, `package.json`, and `certs/` (SSL certs for RDS)
3. Runs `npm install --omit=dev` (production dependencies only)
4. Strips bloat (docs, tests, source maps, changelogs)
5. Zips everything into `deploy.zip`
6. Prints the final zip size
### Upload to AWS
1. Go to your Lambda function → **Code**
2. Click **Upload from** → **.zip file**
3. Select `deploy.zip`
4. Click **Save**
5. Verify **Runtime settings** → **Handler** is set to: `lambda.handler`
### Manual Build (Alternative)
```bash
# Create build directory
mkdir -p dist
cp package.json lambda.js dist/
cp -r src certs dist/
# Install production dependencies
cd dist
npm install --omit=dev --no-package-lock --legacy-peer-deps
# Strip bloat
find node_modules -type f \( -name "*.md" -o -name "*.ts" -o -name "*.map" \
  -o -name "LICENSE" -o -name "README" -o -name "CHANGELOG*" \) -delete
find node_modules -type d \( -name "test" -o -name "tests" -o -name "__tests__" \
  -o -name "docs" -o -name "examples" \) -exec rm -rf {} +
# Zip and clean up
zip -rq ../deploy.zip .
cd .. && rm -rf dist
```
---
## Step 6: Verify Deployment
After deploying, run the following test scripts to verify everything is working. Save each script and run with `node <script-name>.js`.
Replace `YOUR_API_URL` with your API Gateway invoke URL.
### Health Check Test
**File: `tests/test_health.js`**
```javascript
/**
 * Health Check Test
 * Verifies the Lambda function is running and Express is responding.
 */
const API_URL = process.env.API_URL || "https://YOUR_API_URL.execute-api.us-east-1.amazonaws.com";
async function testHealthCheck() {
    console.log("🏥 Testing health check...\n");
    console.log(`   URL: ${API_URL}/health`);
    console.log("   ─────────────────────────────────────");
    try {
        const startTime = Date.now();
        const response = await fetch(`${API_URL}/health`);
        const latency = Date.now() - startTime;
        if (!response.ok) {
            console.error(`   ❌ FAIL: HTTP ${response.status} ${response.statusText}`);
            const text = await response.text();
            console.error(`   Response: ${text}`);
            process.exit(1);
        }
        const data = await response.json();
        console.log(`   ✅ Status:    ${data.status}`);
        console.log(`   ⏱  Uptime:    ${data.uptime?.toFixed(2)}s`);
        console.log(`   🕐 Timestamp: ${data.timestamp}`);
        console.log(`   📡 Latency:   ${latency}ms`);
        if (data.status !== "ok") {
            console.error("\n   ⚠️  WARNING: Status is not 'ok'");
            process.exit(1);
        }
        console.log("\n   ✅ Health check PASSED\n");
    } catch (error) {
        console.error(`   ❌ FAIL: ${error.message}`);
        if (error.cause?.code === "ECONNREFUSED") {
            console.error("   💡 Hint: The server is not reachable. Check your API URL.");
        } else if (error.cause?.code === "ENOTFOUND") {
            console.error("   💡 Hint: DNS lookup failed. Verify the API Gateway URL is correct.");
        } else if (error.message.includes("fetch")) {
            console.error("   💡 Hint: Make sure you are using Node.js 18+ (built-in fetch).");
        }
        process.exit(1);
    }
}
testHealthCheck();
```
---
### Database Connection Test
**File: `tests/test_database.js`**
```javascript
/**
 * Database Connection Test
 * Verifies Lambda can reach the PostgreSQL RDS database.
 */
const API_URL = process.env.API_URL || "https://YOUR_API_URL.execute-api.us-east-1.amazonaws.com";
async function testDatabase() {
    console.log("🗄️  Testing database connection...\n");
    console.log(`   URL: ${API_URL}/debug-db`);
    console.log("   ─────────────────────────────────────");
    try {
        const startTime = Date.now();
        const response = await fetch(`${API_URL}/debug-db`);
        const latency = Date.now() - startTime;
        const data = await response.json();
        if (!response.ok || !data.success) {
            console.error(`   ❌ FAIL: Database connection failed`);
            console.error(`   HTTP Status: ${response.status}`);
            console.error(`   Error:       ${data.error || data.message || "Unknown error"}`);
            console.error(`   Error Code:  ${data.code || "N/A"}`);
            console.error(`   Latency:     ${data.latency || latency + "ms"}`);
            console.error(`   DB ENV Set:  ${data.env_db_set}`);
            // Diagnostic hints
            if (data.code === "ECONNREFUSED") {
                console.error("\n   💡 Hint: Lambda cannot reach the database.");
                console.error("      - Ensure Lambda and RDS are in the same VPC");
                console.error("      - Check Security Group inbound rules (port 5432)");
            } else if (data.code === "ETIMEDOUT") {
                console.error("\n   💡 Hint: Connection timed out.");
                console.error("      - Check VPC subnet routing");
                console.error("      - Ensure RDS Security Group allows Lambda's Security Group");
            } else if (data.code === "28P01") {
                console.error("\n   💡 Hint: Authentication failed.");
                console.error("      - Verify DATABASE_URL username and password");
            } else if (!data.env_db_set) {
                console.error("\n   💡 Hint: DATABASE_URL environment variable is not set in Lambda.");
            }
            process.exit(1);
        }
        console.log(`   ✅ Message:  ${data.message}`);
        console.log(`   🕐 DB Time:  ${data.time_on_db}`);
        console.log(`   📦 Version:  ${data.version}`);
        console.log(`   📡 Latency:  ${data.latency}`);
        console.log(`   🔗 DB ENV:   ${data.env_db_set ? "Set" : "NOT SET"}`);
        console.log(`   🏠 Is Local: ${data.db_is_localhost ? "Yes" : "No (Remote/RDS)"}`);
        console.log("\n   ✅ Database connection PASSED\n");
    } catch (error) {
        console.error(`   ❌ FAIL: ${error.message}`);
        if (error.cause?.code === "ECONNREFUSED") {
            console.error("   💡 Hint: Cannot reach the API. Check your API Gateway URL.");
        } else if (error.message.includes("Unexpected token")) {
            console.error("   💡 Hint: Response is not JSON. The endpoint may not exist or Lambda crashed.");
        }
        process.exit(1);
    }
}
testDatabase();
```
---
### Internet Connectivity Test
**File: `tests/test_internet.js`**
```javascript
/**
 * Internet Connectivity Test
 * Verifies Lambda can reach external services (Firebase, HuggingFace, Pinecone).
 * If this fails, your Lambda likely needs a NAT Gateway for outbound internet.
 */
const API_URL = process.env.API_URL || "https://YOUR_API_URL.execute-api.us-east-1.amazonaws.com";
async function testInternet() {
    console.log("🌐 Testing internet connectivity...\n");
    console.log(`   URL: ${API_URL}/debug-internet`);
    console.log("   ─────────────────────────────────────");
    try {
        const startTime = Date.now();
        const response = await fetch(`${API_URL}/debug-internet`);
        const latency = Date.now() - startTime;
        const data = await response.json();
        if (!response.ok || !data.success) {
            console.error(`   ❌ FAIL: Internet connectivity check failed`);
            console.error(`   HTTP Status: ${response.status}`);
            console.error(`   Error:       ${data.error || data.message || "Unknown error"}`);
            console.error(`   Latency:     ${data.latency || latency + "ms"}`);
            // Diagnostic hints
            if (data.error?.includes("ETIMEDOUT") || data.error?.includes("timeout")) {
                console.error("\n   💡 Hint: Lambda cannot reach the internet. This is a VPC issue.");
                console.error("      FIX: Add a NAT Gateway to your VPC:");
                console.error("      1. Create a NAT Gateway in a PUBLIC subnet");
                console.error("      2. Update the PRIVATE subnet's route table:");
                console.error("         Destination: 0.0.0.0/0 → Target: NAT Gateway");
                console.error("      3. Lambda must be in the PRIVATE subnet");
            } else if (data.error?.includes("ENOTFOUND")) {
                console.error("\n   💡 Hint: DNS resolution failed. Check VPC DNS settings.");
                console.error("      - Enable 'DNS hostnames' and 'DNS resolution' on the VPC");
            }
            process.exit(1);
        }
        console.log(`   ✅ Message:    ${data.message}`);
        console.log(`   🔑 Keys Found: ${data.keys_count} Google signing keys`);
        console.log(`   📡 Latency:    ${data.latency}`);
        console.log("\n   ✅ Internet connectivity PASSED\n");
    } catch (error) {
        console.error(`   ❌ FAIL: ${error.message}`);
        if (error.message.includes("abort") || error.message.includes("timeout")) {
            console.error("   💡 Hint: Request timed out. Lambda may need more time or internet access.");
            console.error("      - Increase Lambda timeout (Configuration → General → Timeout)");
            console.error("      - If in VPC, ensure NAT Gateway is configured");
        }
        process.exit(1);
    }
}
testInternet();
```
---
### Run All Verification Tests
**File: `tests/test_all.js`**
```javascript
/**
 * Run All Deployment Verification Tests
 * Tests health, database, and internet connectivity sequentially.
 *
 * Usage:
 *   API_URL=https://your-api.execute-api.us-east-1.amazonaws.com node tests/test_all.js
 */
const API_URL = process.env.API_URL || "https://YOUR_API_URL.execute-api.us-east-1.amazonaws.com";
const tests = [
    { name: "Health Check", path: "/health" },
    { name: "Database Connection", path: "/debug-db" },
    { name: "Internet Connectivity", path: "/debug-internet" },
];
async function runTest(test) {
    const url = `${API_URL}${test.path}`;
    try {
        const start = Date.now();
        const response = await fetch(url);
        const latency = Date.now() - start;
        const data = await response.json();
        const passed = response.ok && (data.status === "ok" || data.success === true);
        return {
            name: test.name,
            passed,
            latency: `${latency}ms`,
            status: response.status,
            detail: passed
                ? (data.message || data.status || "OK")
                : (data.error || data.message || `HTTP ${response.status}`),
        };
    } catch (error) {
        return {
            name: test.name,
            passed: false,
            latency: "N/A",
            status: "ERR",
            detail: error.message,
        };
    }
}
async function main() {
    console.log("╔══════════════════════════════════════════════════╗");
    console.log("║     DebugFlow — Deployment Verification Suite    ║");
    console.log("╚══════════════════════════════════════════════════╝");
    console.log(`\n   API: ${API_URL}\n`);
    const results = [];
    for (const test of tests) {
        process.stdout.write(`   ⏳ ${test.name}...`);
        const result = await runTest(test);
        results.push(result);
        const icon = result.passed ? "✅" : "❌";
        console.log(`\r   ${icon} ${result.name.padEnd(25)} ${result.latency.padStart(8)}   ${result.detail}`);
    }
    console.log("\n   ──────────────────────────────────────────────");
    const passed = results.filter(r => r.passed).length;
    const total = results.length;
    if (passed === total) {
        console.log(`   🎉 All ${total} tests PASSED!`);
    } else {
        console.log(`   ⚠️  ${passed}/${total} tests passed. ${total - passed} failed.`);
        console.log(`\n   Failed tests:`);
        results.filter(r => !r.passed).forEach(r => {
            console.log(`     ❌ ${r.name}: ${r.detail}`);
        });
    }
    console.log("");
    process.exit(passed === total ? 0 : 1);
}
main();
```
**Run with:**
```bash
API_URL=https://your-api.execute-api.us-east-1.amazonaws.com node tests/test_all.js
```
**Example output:**
```
╔══════════════════════════════════════════════════╗
║     DebugFlow — Deployment Verification Suite    ║
╚══════════════════════════════════════════════════╝
   API: https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com
   ✅ Health Check                  142ms   ok
   ✅ Database Connection           287ms   Database is reachable!
   ✅ Internet Connectivity         315ms   Internet is reachable!
   ──────────────────────────────────────────────
   🎉 All 3 tests PASSED!
```
---
## Connecting to RDS from pgAdmin 4
Use pgAdmin 4 to browse your database tables, run queries, and manage data.
### Step 1: Gather Your RDS Connection Details
Find these in the **AWS RDS Console** → Your database instance → **Connectivity & security**:
| Field | Where to Find It |
|-------|-----------------|
| **Endpoint** | RDS Console → Connectivity & security → Endpoint |
| **Port** | Usually `5432` |
| **Database name** | The name used in your `DATABASE_URL` (e.g., `debugflow_db`) |
| **Username** | The master username set during RDS creation |
| **Password** | The master password set during RDS creation |
### Step 2: Ensure Network Access
Your local machine must be able to reach the RDS instance:
**If RDS is publicly accessible:**
1. Go to **RDS Console** → Your DB → **Modify**
2. Under **Connectivity**, set **Publicly accessible** to **Yes**
3. Go to the **Security Group** attached to your RDS instance
4. Add an **Inbound rule**:
   - **Type**: PostgreSQL
   - **Port**: `5432`
   - **Source**: `My IP` (or your specific IP address)
5. Click **Save rules**
**If RDS is NOT publicly accessible (private subnet only):**
You'll need to use one of these methods:
- **SSH Tunnel** through a bastion host / EC2 instance in the same VPC
- **AWS Client VPN** to connect your machine to the VPC
- **Temporarily** make it publicly accessible (not recommended for production)
### Step 3: Connect in pgAdmin 4
1. Open **pgAdmin 4**
2. Right-click **Servers** → **Register** → **Server...**
3. Fill in the tabs:
**General tab:**
| Field | Value |
|-------|-------|
| **Name** | `DebugFlow RDS` (any label you want) |
**Connection tab:**
| Field | Value |
|-------|-------|
| **Host name/address** | Your RDS endpoint (e.g., `debugflow-db.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com`) |
| **Port** | `5432` |
| **Maintenance database** | `debugflow_db` (your database name) |
| **Username** | Your RDS master username |
| **Password** | Your RDS master password |
| **Save password** | ✅ Check this for convenience |
**SSL tab:**
| Field | Value |
|-------|-------|
| **SSL mode** | `Require` (recommended for RDS) |
4. Click **Save**
### Step 4: Browse Your Tables
Once connected:
1. Expand: **DebugFlow RDS** → **Databases** → **debugflow_db** → **Schemas** → **public** → **Tables**
2. You should see tables like: `users`, `sessions`, `messages`, `usage_logs`, etc.
3. Right-click any table → **View/Edit Data** → **All Rows** to see the data
### Common pgAdmin Issues
| Issue | Solution |
|-------|----------|
| Connection timed out | Check RDS Security Group → add your IP to inbound rules (port 5432) |
| Authentication failed | Verify username/password match what you set during RDS creation |
| SSL connection required | Set SSL mode to `Require` in the SSL tab |
| "No route to host" | RDS is in a private subnet — use an SSH tunnel or make it temporarily public |
---
## Updating Your Frontend
Point your frontend's API base URL to the API Gateway invoke URL:
```typescript
// In your .env.local or .env.production
NEXT_PUBLIC_API_URL=https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com
```
Or in your API configuration file:
```typescript
// frontend/lib/api.ts
const BASE_URL = process.env.NEXT_PUBLIC_API_URL || "http://localhost:4000";
```
Ensure your API Gateway domain (or the `execute-api` URL) is included in the CORS `allowedOrigins` list in `backend/src/app.js`:
```javascript
const allowedOrigins = [
    'http://localhost:3000',
    'https://your-app.vercel.app',
    // Add your API Gateway URL here if needed for browser requests
];
```
---
## Troubleshooting
### CORS Errors
| Error | Cause | Fix |
|-------|-------|-----|
| "Duplicate CORS header" | CORS enabled in both API Gateway AND Express | Disable one — pick API Gateway OR Express, not both |
| "Missing Access-Control-Allow-Origin" | CORS not configured | Enable CORS in API Gateway or ensure Express CORS middleware is active |
| "Preflight (OPTIONS) fails" | API Gateway not handling OPTIONS | Add OPTIONS to allowed methods, or let Express handle preflight |
### Lambda Errors
| Error | Cause | Fix |
|-------|-------|-----|
| "Task timed out" | Timeout too short | Increase Lambda timeout to 60s. Also check API Gateway timeout (30s max for HTTP API) |
| "Module not found" | Missing dependency in deploy.zip | Run `npm install --omit=dev` in the build directory before zipping |
| `Cannot find module './src/app.js'` | File not included in zip | Verify `zip_deploy.sh` copies the `src/` directory |
### Database Errors
| Error | Cause | Fix |
|-------|-------|-----|
| `ECONNREFUSED` | Lambda can't reach RDS | Same VPC + Security Group must allow Lambda's SG on port 5432 |
| `ETIMEDOUT` | Network routing issue | Check subnet route tables. Private subnet needs NAT for internet, but direct route for RDS |
| `28P01` (auth failed) | Wrong credentials | Verify `DATABASE_URL` in Lambda environment variables |
### Internet / External API Errors
| Error | Cause | Fix |
|-------|-------|-----|
| Firebase verification timeout | No outbound internet | Add NAT Gateway: Public subnet → NAT GW → Private subnet route `0.0.0.0/0 → NAT` |
| HuggingFace API unreachable | No outbound internet | Same NAT Gateway fix as above |
| DNS resolution failed | VPC DNS disabled | Enable "DNS hostnames" and "DNS resolution" on the VPC |
---
## Project Structure (Deployed)
```
deploy.zip
├── lambda.js          ← Lambda entry point (handler)
├── package.json       ← Dependencies manifest
├── src/
│   ├── app.js         ← Express app configuration (CORS, middleware, routes)
│   ├── server.js      ← Local dev server (not used in Lambda)
│   ├── routes/        ← API route handlers (chat, sessions, admin, etc.)
│   ├── middleware/     ← Auth (Firebase), rate limiting, error handling
│   ├── services/      ← Chat AI, embeddings, context, RAG
│   ├── db/            ← PostgreSQL (pg pool) + Pinecone connections
│   ├── lib/           ← Firebase Admin SDK initialization
│   └── utils/         ← Logger, retry helpers, environment loader
├── certs/             ← SSL certificates for RDS connections
└── node_modules/      ← Production dependencies only (dev deps excluded)
