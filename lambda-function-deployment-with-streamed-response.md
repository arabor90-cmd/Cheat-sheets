# DebugFlow Backend — AWS Lambda Streaming Deployment Guide

This guide covers deploying the DebugFlow Express backend to **AWS Lambda** using a **Function URL** with **Streamed Response** mode (word-by-word AI replies), and connecting your **RDS PostgreSQL database** via **pgAdmin 4**.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Streaming Lambda Handler (`lambda.js`)](#streaming-lambda-handler-lambdajs)
- [Step 1: Create the Lambda Function](#step-1-create-the-lambda-function)
- [Step 2: Configure Environment Variables](#step-2-configure-environment-variables)
- [Step 3: Create a Function URL (Streaming Mode)](#step-3-create-a-function-url-streaming-mode)
- [Step 4: Configure CORS](#step-4-configure-cors)
- [Step 5: Build & Deploy](#step-5-build--deploy)
- [Step 6: Verify Deployment (Test Scripts)](#step-6-verify-deployment-test-scripts)
- [Connecting to RDS from pgAdmin 4](#connecting-to-rds-from-pgadmin-4)
- [Updating Your Frontend](#updating-your-frontend)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌──────────┐     HTTPS      ┌─────────────────────┐     ┌───────────┐
│ Frontend │ ──────────────▶ │  Lambda Function URL │ ──▶ │  Express  │
│ (Vercel) │ ◀────────────── │  (Streaming Mode)   │ ◀── │   App     │
└──────────┘  (Real-time)    └─────────────────────┘     └───────────┘
                                                               │
                                       ┌───────────────────────┼────────────────┐
                                       ▼                       ▼                ▼
                                  PostgreSQL             Firebase Auth    HuggingFace AI
                                   (AWS RDS)            (Token Verify)    (Streaming SSE)
```

**How it works:**
- **Streaming Mode**: Unlike buffered mode, the response is sent back to the client as chunks are generated. This allows the user to see AI responses word-by-word.
- **`streamifyResponse`**: The official AWS wrapper allows Lambda to bypass the standard 6MB response limit and deliver data incrementally.
- **Custom Adapter**: Bypasses the internal buffering of `serverless-http` for the chat route, piping the AI stream directly to the browser.

---

## Prerequisites

- **AWS Account** with permissions to create Lambda functions and IAM roles.
- **Node.js 20.x or 24.x** (Node.js 24.x is recommended for the best streaming support).
- **pgAdmin 4** installed ([download](https://www.pgadmin.org/download/)).
- **NAT Gateway** (Highly Recommended): If your Lambda is in a VPC, you **must** have a NAT Gateway in a public subnet to allow outbound calls to Firebase/HuggingFace.

---

## Streaming Lambda Handler (`lambda.js`)

This is the code that enables true streaming. It follows the official AWS pattern for Node.js.

**File: `backend/lambda.js`**

```javascript
import { PassThrough } from 'stream';
import http from 'http';
import serverless from 'serverless-http';
import { app } from './src/app.js';

const serverlessHandler = serverless(app);

/**
 * MAIN HANDLER — Following official AWS streaming pattern.
 * The handler MUST be the streamifyResponse wrapper to enable streaming invoke mode.
 */
export const handler = awslambda.streamifyResponse(
    async (event, responseStream, _context) => {
        const method = event.requestContext?.http?.method;
        const path = event.rawPath || event.requestContext?.http?.path || '';
        const isChatPost = path.includes('/api/chat') && method === 'POST';

        if (isChatPost) {
            try {
                // Use custom adapter for word-by-word streaming
                await handleChatStreaming(event, responseStream);
            } catch (err) {
                console.error('[Lambda] Streaming failed, falling back to buffered:', err);
                await handleBuffered(event, responseStream);
            }
        } else {
            // Standard routes (sessions, profile, etc.) use buffered mode
            await handleBuffered(event, responseStream);
        }
    }
);

/**
 * BUFFERED PATH — Standard response handling
 */
async function handleBuffered(event, responseStream) {
    const result = await serverlessHandler(event, {});
    const metadata = {
        statusCode: result.statusCode || 200,
        headers: result.headers || {}
    };
    responseStream = awslambda.HttpResponseStream.from(responseStream, metadata);
    responseStream.write(
        result.isBase64Encoded ? Buffer.from(result.body, 'base64') : (result.body || '')
    );
    responseStream.end();
    await responseStream.finished();
}

/**
 * STREAMING PATH — Bypasses buffering specifically for Chat SSE
 */
async function handleChatStreaming(event, responseStream) {
    const method = event.requestContext?.http?.method || 'POST';
    const rawPath = event.rawPath || '/api/chat';
    const url = rawPath + (event.rawQueryString ? '?' + event.rawQueryString : '');
    const headers = event.headers || {};
    const bodyStr = event.isBase64Encoded ? Buffer.from(event.body || '', 'base64').toString() : (event.body || '');

    const req = new http.IncomingMessage(new PassThrough());
    req.method = method;
    req.url = url;
    req.headers = { ...headers, 'content-length': String(Buffer.byteLength(bodyStr)) };
    req.push(bodyStr);
    req.push(null);

    const res = new http.ServerResponse(req);
    let preambleSent = false;
    const sendPreamble = () => {
        if (preambleSent) return;
        preambleSent = true;
        const metadata = { statusCode: res.statusCode || 200, headers: res.getHeaders() };
        responseStream = awslambda.HttpResponseStream.from(responseStream, metadata);
    };

    res.write = (chunk, encoding, cb) => { sendPreamble(); return responseStream.write(chunk, encoding, cb); };
    res.end = (chunk, encoding, cb) => { 
        sendPreamble(); 
        if (chunk) responseStream.write(chunk, encoding); 
        responseStream.end(); 
        res.emit('finish');
        if (cb) cb(); 
    };
    res.writeHead = (code, msg, hdrs) => { res.statusCode = code; return res; };

    app.handle(req, res);
    await new Promise((res) => { responseStream.on('finish', res); responseStream.on('close', res); });
}
```

---

## Step 1: Create the Lambda Function

1. **Create Function**: Use **Node.js 24.x**.
2. **Architecture**: `x86_64`.
3. **Handler**: Set to `lambda.handler`.
4. **General Configuration**: 
   - Increase **Timeout** to `60-120 seconds`.
   - Set **Memory** to `512 MB` or higher.

---

## Step 2: Configure Environment Variables

Add these in **Configuration → Environment variables**:
- `DATABASE_URL`: `postgresql://user:pass@host:5432/dbname`
- `HUGGINGFACE_API_KEY`: Your key
- `PINECONE_API_KEY`: Your key

---

## Step 3: Create a Function URL (Streaming Mode)

1. Go to **Configuration → Function URL**.
2. Click **Create function URL**.
3. **Auth type**: `NONE`.
4. **Invoke mode**: **RESPONSE_STREAM** (CRITICAL).
5. Click **Save**.

---

## Step 4: Configure CORS

**Do NOT enable CORS in the AWS Console.** 

Because we are using `streamifyResponse`, Express will handle the CORS headers automatically. If you enable it in both places, the browser will reject the response due to duplicate `Access-Control-Allow-Origin` headers.

---

## Step 5: Build & Deploy

Use the existing build script:
```bash
./zip_deploy.sh
```
Upload the resulting `deploy.zip` to the AWS Lambda Console.

---

## Step 6: Verify Deployment (Test Scripts)

Run these scripts locally to verify your deployment. Replace `YOUR_FUNCTION_URL` with your actual endpoint.

### 1. Health Check
```javascript
try {
  const res = await fetch('YOUR_FUNCTION_URL/health');
  const data = await res.json();
  console.log('🏥 Health Check:', data.status === 'ok' ? '✅ PASSED' : '❌ FAILED');
} catch (e) {
  console.error('❌ Health Check Error:', e.message);
}
```

### 2. Database Check
```javascript
try {
  const res = await fetch('YOUR_FUNCTION_URL/debug-db');
  const data = await res.json();
  console.log('🗄️ Database Check:', data.success ? '✅ PASSED' : '❌ FAILED');
  if (!data.success) console.error('Error:', data.error);
} catch (e) {
  console.error('❌ Database Check Error:', e.message);
}
```

### 3. Internet Connectivity (Firebase/AI)
```javascript
try {
  const res = await fetch('YOUR_FUNCTION_URL/debug-internet');
  const data = await res.json();
  console.log('🌐 Internet Check:', data.success ? '✅ PASSED' : '❌ FAILED');
} catch (e) {
  console.error('❌ Internet Check Error:', e.message);
}
```

---

## Connecting to RDS from pgAdmin 4

1. **RDS Security Group**: Add an Inbound Rule for **PostgreSQL (5432)** with Source: `My IP`.
2. **Open pgAdmin 4**: Right-click **Servers → Register → Server**.
3. **General Tab**: Name it `DebugFlow Prod`.
4. **Connection Tab**:
   - **Host**: Your RDS Endpoint (e.g., `xxx.rds.amazonaws.com`).
   - **User**: The master username.
   - **Password**: The master password.
5. **SSL Tab**: Set SSL Mode to `Require`.
6. **Save**: You can now explore your tables in the browser side-panel.

---

## Updating Your Frontend

Ensure your frontend points to the new Function URL:
```typescript
// frontend/lib/api.ts
const BASE_URL = "https://your-unique-id.lambda-url.region.on.aws";
```

---

## Troubleshooting

- **No Word-by-Word Streaming**: Ensure **Invoke Mode** is set to `RESPONSE_STREAM` in the Function URL configuration.
- **Duplicate Headers**: Ensure CORS is disabled in the AWS Lambda Console (let Express handle it).
- **Timeouts**: Increase the Lambda Timeout setting to at least 60 seconds.
- **VPC Outbound Errors**: Ensure your private subnets have a route to a NAT Gateway for internet access.
