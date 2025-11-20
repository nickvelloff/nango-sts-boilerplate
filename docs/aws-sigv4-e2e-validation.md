# AWS SigV4 End-to-End Validation Guide

This guide shows how to validate the AWS SigV4 proxy flow locally using:

- The Nango monorepo (running locally)
- A customer-hosted STS helper (this repo)
- Your own AWS test account (no shared account assumptions)

The goal is to prove that Nango can:

1. Launch locally (UI, Connect, proxy)
2. Fetch temporary credentials from the STS helper using an ExternalId
3. Proxy a signed request to AWS (listing objects in an S3 bucket)

---

## 0. Prerequisites

| Requirement | Details |
| --- | --- |
| Tooling | Node 20+, npm 10+, Docker Desktop, AWS CLI v2, `jq` |
| Local repos | Nango monorepo + this STS helper repo, ideally side by side |
| AWS access | An AWS profile with permissions to create IAM users/roles and an S3 bucket in a test account (e.g., `AWS_PROFILE=sigv4-demo`) |
| Ports | Ensure 3000–3010 and 3055 are free locally. |

Export the base environment you will use throughout the setup:

```bash
export AWS_PROFILE=sigv4-demo            # adjust to your profile
export AWS_REGION=us-east-2              # adjust as needed
export ACCOUNT_ID=<your_aws_account_id>  # e.g., 123456789012
export SIGV4_BUCKET_NAME=<unique-bucket-name> # e.g., nango-sigv4-demo-$(date +%Y%m%d)
```

---

## 1. AWS resources

### 1.1 Create a demo S3 bucket

```bash
aws s3api create-bucket \
  --bucket "${SIGV4_BUCKET_NAME}" \
  --region "${AWS_REGION}" \
  --create-bucket-configuration LocationConstraint="${AWS_REGION}"

# Optional: seed an object used in the proxy test
echo "demo object created $(date -u)" | \
  aws s3 cp - "s3://${SIGV4_BUCKET_NAME}/seed/demo.txt" --region "${AWS_REGION}"

aws s3api list-objects-v2 --bucket "${SIGV4_BUCKET_NAME}" --region "${AWS_REGION}"
```

### 1.2 IAM identities (must exist before running Connect)

1. **STS service IAM user (`NangoSigV4StsUser`)**
   ```bash
   aws iam create-user --user-name NangoSigV4StsUser

   aws iam put-user-policy \
     --user-name NangoSigV4StsUser \
     --policy-name AllowAssumeSigV4DemoRole \
     --policy-document "{
       \"Version\":\"2012-10-17\",
       \"Statement\":[
         {
           \"Effect\":\"Allow\",
           \"Action\":\"sts:AssumeRole\",
           \"Resource\":\"arn:aws:iam::${ACCOUNT_ID}:role/NangoSigV4DemoRole\"
         }
       ]
     }"

   aws iam create-access-key --user-name NangoSigV4StsUser
   ```

   Capture the `AccessKeyId` and `SecretAccessKey`; they feed the STS Docker container via `.env`.

2. **Customer role (`NangoSigV4DemoRole`)**
   ```bash
   cat <<JSON >/tmp/nango-sigv4-trust.json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": { "AWS": "arn:aws:iam::${ACCOUNT_ID}:user/NangoSigV4StsUser" },
         "Action": "sts:AssumeRole",
         "Condition": { "StringEquals": { "sts:ExternalId": "\${externalId}" } }
       }
     ]
   }
   JSON

   aws iam create-role \
     --role-name NangoSigV4DemoRole \
     --assume-role-policy-document file:///tmp/nango-sigv4-trust.json

   aws iam put-role-policy \
     --role-name NangoSigV4DemoRole \
     --policy-name AllowSigV4BucketOps \
     --policy-document "{
       \"Version\":\"2012-10-17\",
       \"Statement\":[
         {\"Effect\":\"Allow\",\"Action\":[\"s3:ListBucket\"],\"Resource\":\"arn:aws:s3:::${SIGV4_BUCKET_NAME}\"},
         {\"Effect\":\"Allow\",\"Action\":[\"s3:GetObject\",\"s3:PutObject\"],\"Resource\":\"arn:aws:s3:::${SIGV4_BUCKET_NAME}/*\"}
       ]
     }"
   ```

   Record the following final values for later:

   | Field | Example / Notes |
   | --- | --- |
   | `STS_SERVICE_ACCESS_KEY_ID` | Output from `create-access-key` |
   | `STS_SERVICE_SECRET_ACCESS_KEY` | Output from `create-access-key` |
   | `ASSUMABLE_ROLE_ARN` | `arn:aws:iam::${ACCOUNT_ID}:role/NangoSigV4DemoRole` |
   | `STS_SERVICE_REGION` | Typically the same as `AWS_REGION` |

> The trust policy enforces ExternalId, so Connect’s generated value must be used. The STS helper just forwards whatever Nango provides; no static value is stored.

### 1.3 Host the CloudFormation template

Quick Create links need a publicly reachable template URL. Upload the template from this repo and (for demo purposes) allow GET/HEAD reads so the Nango UI can hydrate parameters. Lock this down or use signed URLs for any non-demo use.

```bash
aws s3api delete-public-access-block --bucket "${SIGV4_BUCKET_NAME}" --region "${AWS_REGION}"

aws s3 cp infra/cloudformation/s3-readonly.json \
  "s3://${SIGV4_BUCKET_NAME}/templates/s3-readonly.json" \
  --region "${AWS_REGION}"

aws s3api put-bucket-cors --bucket "${SIGV4_BUCKET_NAME}" --region "${AWS_REGION}" --cors-configuration '{
  "CORSRules": [
    {
      "AllowedOrigins": ["*"],
      "AllowedMethods": ["GET", "HEAD"],
      "AllowedHeaders": ["*"],
      "ExposeHeaders": [],
      "MaxAgeSeconds": 300
    }
  ]
}'
```

Template URL to reference later:

```
https://${SIGV4_BUCKET_NAME}.s3.${AWS_REGION}.amazonaws.com/templates/s3-readonly.json
```

---

## 2. Run the STS helper (this repo)

1. Install dependencies:
   ```bash
   npm install
   ```
2. Create `.env` in the repo root:
   ```
   AWS_REGION=${AWS_REGION}
   AWS_ACCESS_KEY_ID=<STS_SERVICE_ACCESS_KEY_ID>
   AWS_SECRET_ACCESS_KEY=<STS_SERVICE_SECRET_ACCESS_KEY>
   STS_SHARED_API_KEY=local-demo-sts-key
   ```
3. Run via Docker (recommended for parity with the Nango UI settings):
   ```bash
   docker compose up --build
   # Service listens on http://localhost:3055/assume-role
   ```

> If you prefer local dev without Docker: `npm run dev` (port defaults to 3000; adjust the Nango STS URL accordingly).

---

## 3. Run Nango locally

1. **Environment variables**
   ```bash
   cd /path/to/nango
   cp .env.example .env
   # adjust DB/server ports if needed
   ```
2. **Dev dependencies (Postgres + Redis)**
   ```bash
   docker compose up -d nango-db nango-redis
   ```
3. **Install & run the monorepo**
   ```bash
   npm install
   npm run dev:watch:apps
   ```
   - Server/API: http://localhost:3003  
   - Dashboard: http://localhost:3000  
   - Connect UI: http://localhost:3009
4. **Initial account**
   - Visit http://localhost:3000
   - Create a root account + environment
   - Generate an **Environment Secret Key** (Settings → Environments) — needed for API calls.

---

## 4. Configure the AWS SigV4 integration (Dashboard)

1. Go to **Integrations → Add Integration** and choose **AWS SigV4**.
2. Set a provider config key (e.g., `aws-sigv4-demo`) and click **Create**.
3. Open the integration → **Settings → General → AWS SigV4 Settings**:
   - **AWS Service:** `s3`
   - **Default Region:** `${AWS_REGION}`
   - **STS Endpoint URL:** `http://localhost:3055/assume-role`
   - **STS Auth:** API key header, header name `x-api-key`, value `local-demo-sts-key`
4. Add a CloudFormation template:
   - Template ID `s3-readonly`
   - Display label `AWS S3 Read Only`
   - Stack name `NangoSigV4Demo`
   - Description `Creates the demo IAM role for the validation flow`
   - Template URL: `https://${SIGV4_BUCKET_NAME}.s3.${AWS_REGION}.amazonaws.com/templates/s3-readonly.json`
   - Click **Load template**, then set parameters:
     - `TrustedAccountId` = `${ACCOUNT_ID}`
     - `TrustedUserName` = `NangoSigV4StsUser`
     - `BucketName` = `${SIGV4_BUCKET_NAME}`
   - Save AWS SigV4 Settings.

The template card should now appear in the integration’s Connect preview.

---

## 5. Create a Connect session (UI-first)

1. In the dashboard, open your `aws-sigv4-demo` integration → **Connect Sessions**.
2. Click **Create Connect Session**.
3. Set:
   - Integration: `aws-sigv4-demo`
   - End user ID `demo-user-001`, email `demo@example.com`
   - Success URL `https://example.com/success`
   - Error URL `https://example.com/error`
4. Create the session and copy the generated link.

---

## 6. Complete the Connect flow

1. Open the Connect link; you will see the template card plus credential form.
2. In another tab, sign in to the AWS console with your test account (the one where you created the bucket/role/user).
3. Click **AWS S3 Read Only** in Connect to open the Quick Create page. The stack name and parameters should be pre-filled.
4. Deploy the stack. After it succeeds, copy the `RoleArn` output (e.g., `arn:aws:iam::${ACCOUNT_ID}:role/NangoSigV4DemoRole`).
5. Return to Connect, paste the IAM Role ARN, keep the region as `${AWS_REGION}` (or override), and submit.
6. On submit, Nango:
   - Generates a unique ExternalId and stores it in the connection config.
   - Calls the STS helper at `http://localhost:3055/assume-role`.
   - Verifies the assumed credentials via `GetCallerIdentity`.
7. After success you land on the Success URL. Find the new connection under the integration and note the **Connection ID** for the proxy test.

---

## 7. Validate the proxy request

Set the values you captured:

```bash
ENV_SECRET=<environment_secret_key_from_dashboard>
CONNECTION_ID=<connection_id_from_dashboard>
```

List the bucket:

```bash
curl -s -X GET "http://localhost:3003/proxy/${SIGV4_BUCKET_NAME}?list-type=2" \
  -H "Authorization: Bearer ${ENV_SECRET}" \
  -H "provider-config-key: aws-sigv4" \
  -H "connection-id: ${CONNECTION_ID}"
```

Expected response (abbreviated):

```json
{
  "data": {
    "Contents": [
      { "Key": "seed/demo.txt", "Size": 44 }
    ]
  },
  "status": 200
}
```

Fetch the seed object (optional):

```bash
curl -s -X GET "http://localhost:3003/proxy/${SIGV4_BUCKET_NAME}/seed/demo.txt" \
  -H "Authorization: Bearer ${ENV_SECRET}" \
  -H "provider-config-key: aws-sigv4" \
  -H "connection-id: ${CONNECTION_ID}"
```

---

## 8. Troubleshooting checklist

- **Connect fails before verifying credentials:** Check the STS container logs to confirm the API key header and AWS credentials are correct.
- **`GetCallerIdentity` errors:** Ensure the IAM trust policy in §1.2 matches the `ExternalId` stored on the connection.
- **Proxy 403s:** Verify the role policy allows `s3:ListBucket` and `s3:GetObject` on `${SIGV4_BUCKET_NAME}`.
- **Docker networking:** If Nango runs inside Docker instead of `npm run dev:watch:apps`, use `http://host.docker.internal:3055/assume-role` for the STS URL so the container can reach the host port.

Following these steps validates the full flow: local UI, Connect magic link, STS credential exchange, and a signed proxied request against your demo S3 bucket.
