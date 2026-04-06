# Serverless Certification Approval System


This guide walks you through manually deploying the resources in the AWS Console.

> **Note:** For a deep dive into the code, architecture, and Step Functions logic, please see [EXPLANATION.md](EXPLANATION.md).

## Prerequisites
- An active AWS Account.
- Access to the AWS Console.

## Step 1: Create DynamoDB Table

1. Go to the **DynamoDB** console.
2. Click **Create table**.
3. **Table name**: `CertificationRequests`
4. **Partition key**: `requestId` (String)
5. Leave other settings as default and click **Create table**.
6. Wait for the table to become active.

## Step 2: Create IAM Role for Lambda

1. Go to the **IAM** console.
2. Click **Roles** -> **Create role**.
3. Select **AWS service** and choose **Lambda**.
4. Click **Next** for permissions.
5. Add the following permissions policies (search and check):
   - `AmazonDynamoDBFullAccess` (For simplicity in this demo)
   - `AWSStepFunctionsFullAccess` (For simplicity)
   - `CloudWatchLogsFullAccess`
6. Click **Next**.
7. **Role name**: `CertificationLambdaRole`
8. Click **Create role**.

## Step 3: Create Lambda Functions

You will create 4 functions. For each function:
- **Runtime**: Python 3.4 (or latest)
- **Architecture**: x86_64
- **Execution role**: Use an existing role -> `CertificationLambdaRole`

### 3.1 Function: `SubmitRequestFunction`
1. Create function named `SubmitRequestFunction`.
2. Copy code from [src/submit_request.py](src/submit_request.py).
3. **Configuration** -> **Environment variables**:
   - Key: `STATE_MACHINE_ARN`
   - Value: *(Leave blank for now, we will update this later)*

### 3.2 Function: `NotifyManagerFunction`
1. Create function named `NotifyManagerFunction`.
2. Copy code from [src/notify_manager.py](src/notify_manager.py).

### 3.3 Function: `HandleApprovalFunction`
1. Create function named `HandleApprovalFunction`.
2. Copy code from [src/handle_approval.py](src/handle_approval.py).

### 3.4 Function: `CheckStatusFunction`
1. Create function named `CheckStatusFunction`.
2. Copy code from [src/check_status.py](src/check_status.py).
3. **Configuration** -> **Environment variables**:
   - Key: `TABLE_NAME`
   - Value: `CertificationRequests`

## Step 4: Create Step Functions State Machine

1. Go to the **Step Functions** console.
2. Click **Create state machine**.
3. Select **Blank** template or **Write your workflow in code**.
4. In the code editor, replace everything with the content of [step-functions-definition.json](step-functions-definition.json).
5. **CRITICAL**: You must update the placeholders in the JSON with your actual resource names/ARNs:
   - Replace `${DynamoDBTableName}` with `CertificationRequests` (appears twice).
   - Replace `${NotifyManagerFunctionName}` with `NotifyManagerFunction` (or the full ARN if cross-account, but function name works if in same account).
6. Click **Next**.
7. **Name**: `ApprovalStateMachine`
8. **Permissions**: Select **Create a new role** (Step Functions will automatically add permission to invoke Lambda and access DynamoDB based on your definition).
9. Click **Create state machine**.
10. **Copy the ARN** of the created State Machine (e.g., `arn:aws:states:us-east-1:123456789012:stateMachine:ApprovalStateMachine`).

## Step 5: Update Lambda Configuration

1. Go back to **Lambda** console -> `SubmitRequestFunction`.
2. **Configuration** -> **Environment variables**.
3. Update `STATE_MACHINE_ARN` with the ARN you just copied.
4. Click **Save**.

## Step 6: Create API Gateway

1. Go to **API Gateway** console.
2. Click **Create API** -> **HTTP API** (Build).
3. **API name**: `CertificationAPI`.
4. **Integrations**: You can add them now or later. Let's add them now.
   - Integration targets: Lambda
   - Choose `SubmitRequestFunction`, `HandleApprovalFunction`, `CheckStatusFunction`.
5. **Configure routes**:
   - `POST /request` -> `SubmitRequestFunction`
   - `POST /approval` -> `HandleApprovalFunction`
   - `GET /request/{requestId}` -> `CheckStatusFunction`
6. Click **Next** (Stages) -> Leave as `$default`.
7. Click **Create**.
9. Note the **Invoke URL** (e.g., `https://xyz.execute-api.us-east-1.amazonaws.com`).


## Verification Steps

1. **Submit a Request**:
   ```bash
   curl -X POST https://<YOUR-API-URL>/request \
     -H "Content-Type: application/json" \
     -d '{"name": "Alice", "course": "AWS Certified Developer", "cost": 150}'
   ```
   *Response*: `{"requestId": "uuid...", "executionArn": "..."}`

2. **Check Logs for Token**:
   - Go to CloudWatch Logs for `NotifyManagerFunction`.
   - Find the log entry `APPROVAL TOKEN: ...`. Copy the long token string.

3. **Check Status (Pending)**:
   ```bash
   curl https://<YOUR-API-URL>/request/<REQUEST-ID>
   ```
   *Response*: `{"status": "PENDING", ...}`

4. **Approve Request**:
   ```bash
   curl -X POST https://<YOUR-API-URL>/approval \
     -H "Content-Type: application/json" \
     -d '{"requestId": "<REQUEST-ID>", "decision": "APPROVED", "taskToken": "<PASTE-TOKEN-HERE>"}'
   ```

5. **Verify Status (Approved)**:
   ```bash
   curl https://<YOUR-API-URL>/request/<REQUEST-ID>
   ```
   *Response*: `{"status": "APPROVED", ...}`

## Troubleshooting

### "Task Timed Out" Error
If you receive a `{"error": "Task Timed Out"}` response when approving:
- **Cause**: The Step Functions execution took too long and timed out. This often happens if you accidentally created an **Express Workflow** (which has a 5-minute limit) instead of a **Standard Workflow**.
- **Fix**: 
    1. Ensure your State Machine is **Standard** type (default for long-running processes).
    2. Submit a new request and approve it promptly.

## Approval UI (HTML) Setup

This project includes a simple manager UI file: [approval-ui.html](approval-ui.html). It lets you approve or reject without running curl commands manually.

### 1) Host the HTML in S3 Static Website
1. Create or open your S3 bucket for static hosting.
2. Enable **Static website hosting**.
3. Upload `approval-ui.html` as `index.html`.
4. Make objects publicly readable (for demo only).
5. Open the website endpoint in browser.

### 2) Configure API Endpoint in HTML
Update this line in `approval-ui.html`:

```js
const DEFAULT_CALLBACK_API = "https://mql8w7e6h1.execute-api.us-east-1.amazonaws.com/approval";
```

Use your real API URL. If your API uses a stage path, use `/prod/approval` (or your stage name).

### 3) NotifyManagerFunction Environment Variable
In Lambda `NotifyManagerFunction`, set:
- `APPROVAL_UI_BASE_URL` = your S3 website URL (for example `http://certificationapprovalworkflow.s3-website-us-east-1.amazonaws.com/index.html`)

Then CloudWatch logs print a one-click approval link with:
- `taskToken`
- `requestId`
- `name`
- `course`
- `cost`

### 4) API Gateway CORS (Required for Browser Button)
For route `POST /approval`:
- Allow origins: `*` (or your S3 website origin)
- Allow methods: `POST, OPTIONS`
- Allow headers: `Content-Type`
- Deploy API after changes

### 5) Test Flow with UI
1. Submit request using `POST /request`.
2. Open CloudWatch logs for `NotifyManagerFunction`.
3. Open the generated approval URL.
4. Click **Approve** or **Reject**.
5. Verify final status using `GET /request/{requestId}`.

 <img width="1554" height="784" alt="image" src="https://github.com/user-attachments/assets/10ba0001-cfce-4c68-8342-f219241eb49b" />

  
  <img width="886" height="573" alt="image" src="https://github.com/user-attachments/assets/8edc1321-e4fb-42ee-b8c8-a98380448d2b" />

  <img width="884" height="573" alt="image" src="https://github.com/user-attachments/assets/4b5ba1e0-f4a0-4670-89c3-97c43e989d9d" />



### Common UI Issues
- **Error: Failed to fetch**: Usually CORS or wrong callback URL path.
- **403 Forbidden on S3**: Static website/public read not configured.
- **Missing API URL**: `DEFAULT_CALLBACK_API` is still empty or placeholder.
