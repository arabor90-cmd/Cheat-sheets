# DebugFlow Backend — AWS Lambda Deployment Guide

This guide covers deploying the DebugFlow Express backend to **AWS Lambda** using a **Function URL** with **Buffered Response** mode.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Lambda Handler (`lambda.js`)](#lambda-handler-lambdajs)
- [Step 1: Create the Lambda Function](#step-1-create-the-lambda-function)
- [Step 2: Configure Environment Variables](#step-2-configure-environment-variables)
- [Step 3: Create a Function URL](#step-3-create-a-function-url)
- [Step 4: Configure CORS](#step-4-configure-cors)
- [Step 5: Build & Deploy](#step-5-build--deploy)
- [Step 6: Verify Deployment](#step-6-verify-deployment)
- [Updating Your Frontend](#updating-your-frontend)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌──────────┐     HTTPS      ┌─────────────────────┐     ┌───────────┐
│ Frontend │ ──────────────▶ │  Lambda Function URL │ ──▶ │  Express  │
│ (Vercel) │ ◀────────────── │  (Buffered Mode)     │ ◀── │   App     │
└──────────┘                 └─────────────────────┘     └───────────┘
                                                               │
                                       ┌───────────────────────┼────────────────┐
                                       ▼                       ▼                ▼
                                  PostgreSQL             Firebase Auth     HuggingFace AI
                                   (RDS/Neon)            (Token Verify)    (Chat/Embeddings)
```

**How it works:**
- `serverless-http` wraps your Express app so it runs inside Lambda
- Lambda Function URL provides a public HTTPS endpoint (no API Gateway needed)
- **Buffered mode**: Lambda executes the full request and returns the complete response
- Express handles all middleware (CORS, authentication, rate limiting) internally

---

## Prerequisites

- **AWS Account** with permissions to create Lambda functions, IAM roles, and VPC resources
- **Node.js 20+** installed locally
- **AWS CLI** (optional, for CLI-based deployment)
- A **PostgreSQL database** accessible from Lambda (e.g., AWS RDS, Neon, Supabase)
- A **Firebase project** with Authentication enabled
- **HuggingFace API key** (for AI chat and embeddings)

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
- `callbackWaitsForEmptyEventLoop = false` prevents Lambda from waiting for idle DB connections
- `serverless-http` converts Lambda events into Express `req`/`res` objects automatically
- All Express middleware (CORS, auth, rate limiting) works without modification

---

## Step 1: Create the Lambda Function

### Via AWS Console

1. Go to **AWS Lambda** → **Create function**
2. Choose **Author from scratch**
3. Configure:
   - **Function name**: `debugflow-backend` (or your preferred name)
   - **Runtime**: `Node.js 20.x` or higher
   - **Architecture**: `x86_64`
   - **Handler**: `lambda.handler`
4. Under **Advanced settings**:
   - Check **Enable VPC** if your database requires it
   - Select the appropriate **VPC**, **Subnets** (private), and **Security Groups**
5. Click **Create function**

### Post-Creation Settings

1. **General Configuration** → **Edit**:
   - **Memory**: `512 MB` (recommended minimum)
   - **Timeout**: `60 seconds` (AI responses can take time)
   - **Ephemeral storage**: `512 MB` (default is fine)

2. **Execution Role**: Ensure the Lambda role has:
   - `AWSLambdaBasicExecutionRole` (CloudWatch Logs)
   - `AWSLambdaVPCAccessExecutionRole` (if using VPC)

> **Note**: If your Lambda is in a VPC, you need a **NAT Gateway** in a public subnet for outbound internet access (required for Firebase, HuggingFace, and Pinecone API calls).

---

## Step 2: Configure Environment Variables

Go to **Configuration** → **Environment variables** → **Edit** and add:

### Required Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | `postgresql://user:pass@host:5432/dbname` |
| `HUGGINGFACE_API_KEY` | HuggingFace Inference API key | `hf_xxxxxxxx` |
| `PINECONE_API_KEY` | Pinecone vector DB API key | `pcsk_xxxxxxxx` |

### Firebase (Required for Auth)

Your Firebase service account credentials should be available. The backend uses `firebase-admin` which reads from a service account JSON file or environment variables configured in your `firebaseAdmin.js`.

### Optional Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `HUGGINGFACE_CHAT_MODEL` | Chat model name | `deepseek-ai/DeepSeek-V3.1:novita` |
| `HUGGINGFACE_CHAT_TIMEOUT_MS` | Chat timeout in ms | `8000` |
| `HUGGINGFACE_EMBEDDING_MODEL` | Embedding model | `sentence-transformers/all-MiniLM-L6-v2` |
| `PINECONE_INDEX_NAME` | Pinecone index name | `debugflow-logs` |
| `SUPER_USER_ID` | Firebase UID for initial super user | — |
| `SUPER_USER_EMAIL` | Email for initial super user | — |
| `LOG_LEVEL` | Logging level | `info` |

---

## Step 3: Create a Function URL

1. Go to your Lambda function → **Configuration** → **Function URL**
2. Click **Create function URL**
3. Configure:
   - **Auth type**: `NONE` (your Express app handles Firebase authentication)
   - **Invoke mode**: `BUFFERED`
4. Click **Save**

You'll receive a URL like:
```
https://xxxxxxxxxx.lambda-url.us-east-1.on.aws/
```

---

## Step 4: Configure CORS

**Important**: Express handles CORS internally. **Do NOT configure CORS in the AWS Console** — this causes duplicate headers and errors.

1. Go to **Function URL** → **Edit**
2. Leave **CORS** section **empty/disabled**
3. Save

Your Express `app.js` already has CORS configured:
```javascript
app.use(cors({
    origin: ['http://localhost:3000', 'https://your-app.vercel.app'],
    credentials: true,
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH', 'OPTIONS'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-Requested-With'],
}));
```

> **⚠️ Common Mistake**: Enabling CORS in both the AWS Console AND Express causes `Access-Control-Allow-Origin` to appear twice, which browsers reject.

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

This script:
1. Creates a clean `dist/` directory
2. Copies `lambda.js`, `src/`, `package.json`, and `certs/`
3. Installs production dependencies only (`npm install --omit=dev`)
4. Strips unnecessary files (docs, tests, source maps)
5. Zips everything into `deploy.zip`

### Upload to AWS

1. Go to your Lambda function → **Code**
2. Click **Upload from** → **.zip file**
3. Select `deploy.zip`
4. Click **Save**
5. Verify the **Handler** is set to `lambda.handler` under **Runtime settings**

### Manual Build (Alternative)

If you prefer not to use the script:

```bash
# Create build directory
mkdir -p dist && cp package.json lambda.js dist/ && cp -r src dist/

# Install production deps
cd dist && npm install --omit=dev --legacy-peer-deps

# Zip
zip -rq ../deploy.zip .
cd .. && rm -rf dist
```

---

## Step 6: Verify Deployment

### Health Check

```bash
curl https://YOUR-FUNCTION-URL.lambda-url.us-east-1.on.aws/health
```

Expected response:
```json
{
    "status": "ok",
    "uptime": 0.123,
    "timestamp": "2026-02-14T22:00:00.000Z"
}
```

### Database Check

```bash
curl https://YOUR-FUNCTION-URL.lambda-url.us-east-1.on.aws/debug-db
```

### Internet Connectivity Check

```bash
curl https://YOUR-FUNCTION-URL.lambda-url.us-east-1.on.aws/debug-internet
```

If this fails, your Lambda likely needs a **NAT Gateway** for outbound internet access.

---

## Updating Your Frontend

Point your frontend's API base URL to the Lambda Function URL:

```typescript
// frontend/lib/api.ts or .env
const BASE_URL = "https://YOUR-FUNCTION-URL.lambda-url.us-east-1.on.aws";
```

Ensure your Lambda Function URL domain is included in the CORS `allowedOrigins` list in `app.js`.

---

## Troubleshooting

### "Missing or duplicate CORS headers"
- **Cause**: CORS is enabled in BOTH the AWS Console and Express
- **Fix**: Disable CORS in the AWS Function URL settings. Let Express handle it.

### "Task timed out after X seconds"
- **Cause**: Lambda timeout is too short for AI responses
- **Fix**: Increase timeout to `60 seconds` in General Configuration

### "ECONNREFUSED" or database connection errors
- **Cause**: Lambda can't reach the database
- **Fix**: Ensure Lambda is in the same VPC as your database, or that the database allows external connections

### "Firebase verification timeout"
- **Cause**: Lambda can't reach Firebase servers (no internet)
- **Fix**: Add a NAT Gateway to your VPC for outbound internet access

### Cold starts are slow
- **Cause**: Lambda initializes a new container on first request after idle
- **Tips**:
  - Increase memory (also increases CPU proportionally)
  - Use [Provisioned Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html) for warm starts (additional cost)

---

## Project Structure (Deployed)

```
deploy.zip
├── lambda.js          ← Lambda entry point (handler)
├── package.json
├── src/
│   ├── app.js         ← Express app configuration
│   ├── routes/        ← API route handlers
│   ├── middleware/     ← Auth, rate limiting, error handling
│   ├── services/      ← Chat AI, embeddings, context
│   ├── db/            ← PostgreSQL + Pinecone connections
│   ├── lib/           ← Firebase Admin SDK setup
│   └── utils/         ← Logger, retry helpers
├── certs/             ← SSL certificates (if needed)
└── node_modules/      ← Production dependencies only
```
