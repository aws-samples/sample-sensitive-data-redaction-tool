# Sample Sensitive Data Redaction Tool

Upload a `.pdf` or `.docx` file and automatically detect and black out personally identifiable information (PII), both in the document body and in text inside embedded images.

> **Important: this is sample code for demonstration and learning purposes only. It is not production-ready.** It has not been hardened, load-tested, or reviewed for compliance with any regulatory regime. Detection is probabilistic: it can miss PII and it can flag content that is not PII. Always have a human review redacted output before relying on it. You are responsible for evaluating, securing, and testing this code before using any part of it with real data. Deploying it creates AWS resources that incur charges.

- **Backend:** AWS serverless (API Gateway, Lambda, DynamoDB, S3, SQS, Cognito). The core is a Redaction Agent running on **Amazon Bedrock AgentCore Runtime**. Deployed with AWS CDK (Python).
- **Frontend:** React, TypeScript and Vite, calling the backend API over HTTPS. The UI is available in Simplified Chinese (default) and English, switchable from the top bar.
- **Pipeline:** a fixed, deterministic sequence (the LLM deliberately does not plan the steps): extract, image OCR, page OCR (`.pdf`), match (regex, Bedrock Guardrails and an LLM as three detection sources), redact, verify, store, update. The entry point is `handle_job` in `backend/agent/orchestration.py`.

## Repository layout

```
backend/
  app.py                    CDK entry point (stack: PiiDocumentRedactionPocStack)
  infra/redaction_stack.py  All AWS resource definitions
  lambdas/                  API and dispatch Lambda functions
  agent/                    Redaction Agent and its tools, run in AgentCore Runtime
  shared/                   Shared backend models and utilities
  scripts/                  Agent bundle build/sync scripts
  agent_bundle/             Build output (gitignored); CDK deploys its package_clean/
  tests/                    Backend tests
docs/                       Least-privilege IAM policies for deployment
frontend/                   React frontend
```

## Prerequisites

| Item | Requirement |
| --- | --- |
| Operating system | **Linux** (the agent bundle must be built on a Linux host). It does not work on Windows (see Troubleshooting) |
| Python | 3.12 (`python3.12` on PATH) |
| Node.js | `^22.22.2` or `^24.15.0` (see `engines` in `frontend/package.json`) |
| AWS CDK CLI | `npm install -g aws-cdk` |
| AWS CLI | Configured credentials; see "Minimum IAM permissions for deployment" below |
| Region | **`us-west-2`** |
| CDK bootstrap | `cdk bootstrap` run once for the target account and region |
| Bedrock model access | Anthropic Claude Haiku 4.5 and Claude Sonnet 4.5 enabled in us-west-2 (uses the `us.` cross-region inference profiles) |

> **Pin the region first.** The CDK stack does not hardcode a region; it uses the current environment's. If your shell still has `AWS_DEFAULT_REGION=us-east-1`, the whole stack is created in the wrong region. Before every deployment run:
>
> ```bash
> export AWS_REGION=us-west-2
> export AWS_DEFAULT_REGION=us-west-2
> aws sts get-caller-identity   # confirm the right account
> ```

### Minimum IAM permissions for deployment

`docs/` contains a set of optional least-privilege policies, split by purpose:

- **Daily deploy policy** [`docs/least-privilege-deploy-policy.json`](docs/least-privilege-deploy-policy.json): stays attached to the deploy identity (IAM user or role) that runs `cdk deploy` and `cdk diff`. It can only assume the CDK deploy, file-publishing and lookup roles, read this application's stack, and run admin recovery on this application's user pool. It cannot create any resource and cannot change IAM.
- **Bootstrap policy** [`docs/least-privilege-bootstrap-policy.json`](docs/least-privilege-bootstrap-policy.json): one-time. Attach it only while running `cdk bootstrap` and detach it right after; attach it again only for a later re-bootstrap (for example, a CDK upgrade). It can change the CDK bootstrap roles, so it must not stay attached.
- **Execution role policies and permissions boundary**: the two policies attached to the CloudFormation execution role in a separate bootstrap ([`least-privilege-cfn-exec-policy.json`](docs/least-privilege-cfn-exec-policy.json) and [`least-privilege-cfn-exec-iam-policy.json`](docs/least-privilege-cfn-exec-iam-policy.json)), and the permissions boundary [`least-privilege-app-boundary-policy.json`](docs/least-privilege-app-boundary-policy.json) (`PiiRedactionAppBoundary`). The execution role can only create IAM roles that carry this boundary, and the boundary limits which services and resources the application roles can reach at most.

Attach the daily deploy policy:

```bash
aws iam create-policy --policy-name pii-redaction-deploy \
  --policy-document file://docs/least-privilege-deploy-policy.json
aws iam attach-user-policy --user-name <YOUR_DEPLOY_USER> --policy-arn <POLICY_ARN>
# or: aws iam attach-role-policy --role-name <YOUR_DEPLOY_ROLE> --policy-arn <POLICY_ARN>
```

> **Note:** the CloudFormation execution role created by a default `cdk bootstrap` has `AdministratorAccess` and is shared by every CDK application in the same account and Region. To narrow resource-creation permissions to this application as well, bootstrap once with a separate qualifier: create the three managed policies, attach the bootstrap policy temporarily and run `cdk bootstrap` (passing both execution policies), detach the bootstrap policy, and add the qualifier and the `@aws-cdk/core:permissionsBoundary` contexts to `backend/cdk.json`. The full steps, the evidence and the parts that are not verified are in [`docs/least-privilege-deploy-policy.md`](docs/least-privilege-deploy-policy.md).

### Data residency and compliance (read before deploying)

This application processes customer documents that have not been redacted yet. The application itself keeps this data only briefly (see the table below), but the items that follow depend on **account-level configuration** that the application cannot control. Decide how to handle them according to your own compliance requirements.

#### 1. Bedrock model invocation logging (needs the most attention)

To detect PII and to run image OCR, the application sends document text and page or embedded images to Bedrock models (Claude Haiku 4.5 and Sonnet 4.5). If the target account has [Bedrock model invocation logging](https://docs.aws.amazon.com/bedrock/latest/userguide/model-invocation-logging.html) enabled in that Region with text or image delivery turned on, **the complete input and output of every call (that is, the unredacted text and images) is written to CloudWatch Logs or S3**. The retention period is whatever that logging configuration says, and by default logs never expire.

This is an account and Region level setting. It applies to every application in the account and cannot be excluded by role or by application. Check it before deploying:

```bash
aws bedrock get-model-invocation-logging-configuration --region us-west-2
# Empty response: logging is off
# A loggingConfig with textDataDeliveryEnabled / imageDataDeliveryEnabled set to true:
#   document content processed by this application is recorded in cloudWatchConfig.logGroupName or s3Config.bucketName
```

If it is enabled, your options are below. All of them affect other Bedrock applications in the same account, so check with the account owner first:

- Deploy this application into a **separate account that has invocation logging off** (recommended for production).
- Turn invocation logging off, or turn off only text and image delivery.
- Keep invocation logging, but set a short retention period on the target log group (or a lifecycle rule on the target S3 bucket) and restrict who can read it.

Note that turning logging off or shortening retention does not delete only the history this application produced, and CloudWatch Logs cannot delete selectively by caller. If you have already tested with real documents, assess the existing records as well.

#### 2. Amazon Textract and AI services opt-out policies

Page OCR sends page images to Textract synchronously (`DetectDocumentText`, with the bytes passed directly), and the application stores nothing in Textract. However, under the AWS service terms, AI services such as Textract may store and use input content to improve the service unless an [AI services opt-out policy](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ai-opt-out.html) is configured in AWS Organizations. If this matters to you, have your organization administrator confirm or configure it. Bedrock does not use customer inputs or outputs to train models.

#### 3. Data retention in the application

| Location | Content | Retention |
| --- | --- | --- |
| Source bucket | Uploaded original documents | Deleted when the job completes; anything left over expires after 1 day through a lifecycle rule |
| Output bucket | Redacted documents; pending-review detection results `detections.json` (contains PII fragments) | `detections.json` is deleted when the job leaves the review state; the rest expires after 1 day through a lifecycle rule |
| `RedactionJobs` table | Job status and file names (no document content) | DynamoDB TTL of 1 day; the table's stream retains records for 24 hours |
| `RedactionRules` table | Redaction rules configured by administrators | Kept indefinitely, maintained by administrators. If a rule contains a specific person's name or a number, the rule itself is PII |
| `RuleOperations` table | Rule operation records (may contain a rule pattern) | DynamoDB TTL of 1 day |
| Lambda and AgentCore logs | Job ID, status, exception type (the code does not log document content) | CloudWatch log groups keep logs for 1 day (`LOG_RETENTION`) and are deleted with the stack. After `cdk destroy` a few empty `/aws/lambda/PiiDocumentRedactionPocSt-*` log groups may remain (the functions behind the custom resources are invoked one last time during deletion). They contain no document content and can be deleted manually |
| API access logs | One JSON line per request: request ID, time, source IP, the caller's Cognito user ID (`sub`), method, route, status code and response size (request and response bodies are not logged) | Log group `/aws/apigateway/PiiDocumentRedactionPocApi-access` keeps logs for 1 day (`LOG_RETENTION`) and is deleted with the stack |

Two notes:

- S3 lifecycle expiry and DynamoDB TTL both run asynchronously, so actual deletion can happen one to two days later than the configured 1 day. If you need a stricter upper bound, add a scheduled cleanup.
- Redacted files downloaded through the frontend and the original documents on users' machines are outside what this application manages.

## Known limitations (read before any real use)

This sample shows one way to build a document redaction workflow. It deliberately leaves out several controls that a production deployment needs. Review and close these gaps for your own environment before you process real documents.

**Sign-in and access**

- Cognito MFA is off, and Cognito threat protection (advanced security) is not enabled. Turn both on for real users.
- The app client uses the `USER_PASSWORD_AUTH` flow: the password is sent to Cognito over TLS rather than through the SRP protocol.
- Access tokens stay valid for up to 1 hour after a user is demoted or deleted (see section 1.5.1).
- The frontend keeps tokens in `sessionStorage`. If you change the frontend, avoid introducing cross-site scripting (XSS) issues, because injected script could read the tokens.

**Abuse and availability**

- There is no AWS WAF web ACL and no per-user quota. The only limit is stage-wide API throttling (10 requests per second, burst 20). A signed-in user can generate Bedrock, Textract and Lambda costs up to that rate.
- Uploads are checked by file extension (`.pdf` or `.docx`), by their first bytes (a ZIP signature for `.docx`, `%PDF-` within the first 1024 bytes for `.pdf`) and by size (at most 4 MB). Files are not scanned for malware.
- Custom regex rules have no execution timeout. Rule authors are trusted administrators; a badly written pattern (for example, nested repetition) can slow processing down or make a job time out.

**Data protection and audit**

- Detection is probabilistic. It can miss PII, so a person must review every redacted document before it is shared.
- S3 buckets and DynamoDB tables use AWS owned or AWS managed encryption keys, not customer managed KMS keys. S3 versioning, S3 server access logging and DynamoDB point-in-time recovery are off.
- All logs, including API access logs, are kept for 1 day to limit how long data stays around. If you need a longer audit trail, raise the retention or export the logs, and check that no document content is written to them.
- The Lambda functions and the AgentCore Runtime run outside a VPC (AgentCore network mode `PUBLIC`). They reach AWS services over public AWS endpoints with TLS.

**Deployment permissions**

- A default `cdk bootstrap` gives the CloudFormation execution role `AdministratorAccess` (see "Minimum IAM permissions for deployment").

## 1. Deploy the backend

Run all commands from the `backend/` directory.

### 1.1 Create a virtualenv and install dependencies (first time only)

```bash
cd backend
python3.12 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
```

`requirements.txt` includes the CDK libraries (`aws-cdk-lib`, `constructs`), `python-docx`, `strands-agents`, `bedrock-agentcore`, and the PDF dependencies listed below.

#### PDF dependencies and licenses

The manifests `backend/requirements.txt`, `requirements-dev.txt` and `agent-runtime-requirements.txt` are the source of truth for versions. The full third-party license texts are in `THIRD-PARTY-LICENSES` at the repository root.

| Package | Exact version | License | Purpose |
| --- | --- | --- | --- |
| `pdfminer.six` | 20260107 | MIT | PDF text and character-coordinate extraction |
| `pikepdf` | 10.17.0 | MPL-2.0 (the wheel bundles qpdf, Apache-2.0, and others) | PDF structure read/write and content-stream redaction |
| `pypdfium2` | 5.14.0 | BSD-3-Clause (licenses of PDFium and its bundled components are in `THIRD-PARTY-LICENSES`) | PDF page rendering |
| `Pillow` | 13.0.0 | MIT-CMU (HPND) | Image decoding, masking and re-encoding |
| `pypdf` | 6.21.4 | BSD-3-Clause | Development and test only; not in the bundle |
| `hypothesis` | 6.112.1 | MPL-2.0 | Development and test only; not in the bundle |

About MPL-2.0: `pikepdf` and `hypothesis` are both used **unmodified** (installed from PyPI, with no changes to their source and no redistribution of modified versions), so the file-level copyleft obligations of MPL-2.0 do not extend to this project's code.

### 1.2 Run the tests (optional, recommended)

```bash
.venv/bin/python -m pytest tests -q
```

`test_llm_cjk_recall.py` needs real model calls and is skipped by default.

### 1.3 Build the agent bundle

AgentCore Runtime deploys `backend/agent_bundle/package_clean/`. It consists of third-party dependencies (Linux ARM64 / CPython 3.12 wheels) plus this repository's `agent/`, `shared/` and the root `main.py`. **This directory is not committed. Rebuild it before every deployment and after any change under `agent/` or `shared/`:**

```bash
python scripts/build_agent_bundle.py   # run in backend/, using the .venv python
```

The script is idempotent and works in three steps: download aarch64 wheels, install them into `package/`, then strip `*.dist-info`, `bin` and `__pycache__` to produce `package_clean/`. Finally `scripts/sync_agent_bundle.py` overlays the first-party code. Runtime dependencies are listed in `agent-runtime-requirements.txt` (do **not** add `aws-cdk-lib` to it).

Quick sanity checks after the build:

```bash
ls agent_bundle/package_clean | head
find agent_bundle/package_clean \( -name '*.pyd' -o -name '*.dll' \) | wc -l   # expect 0
cmp agent_bundle/package_clean/main.py agent/main.py && echo OK                # expect OK
```

> The build targets Linux ARM64 / CPython 3.12, and the script must run on a **Linux host** (Windows is not supported; on macOS use a Linux host or container). The script uses the platform tags `manylinux2014_aarch64` and `manylinux_2_28_aarch64` and does **not** pass `--abi`: the abi3 wheel of `cryptography` and the `py3-none` wheel of `pypdfium2` are matched through the platform tags, and an explicit `--abi` would stop them matching. Do not "simplify" the pip arguments in the script.

### 1.4 Deploy

You must provide the **email of the first administrator** at deploy time. After the deployment finishes, Cognito sends an invitation email (with a temporary password) to that address, so no manual account creation step is needed. The optional `appUrl` is the frontend address and is written into the invitation email.

```bash
export AWS_REGION=us-west-2 AWS_DEFAULT_REGION=us-west-2
cdk diff -c adminEmail=admin@example.com            # review changes; a first deployment lists every resource as new
cdk deploy \
  -c adminEmail=admin@example.com \
  -c appUrl=https://redaction.example.com             # optional
```

You can use the environment variables `ADMIN_EMAIL` and `APP_URL` instead of `-c`. Without an administrator email, `cdk synth`, `diff` and `deploy` fail immediately. On later redeployments keep using the same email: that step only makes sure this administrator exists. It does not resend the invitation and does not delete or demote anyone. A different email adds one more administrator.

Notes:

- Run from `backend/`; CDK reads the bundle through the relative path `agent_bundle/package_clean`.
- For an **existing** environment this should be an **UPDATE** (Lambda code and the AgentCore Runtime artifact change). If `cdk diff` shows dozens of resources as `[+]`, check the region and account before continuing.
- Confirm the stack status afterwards:

```bash
aws cloudformation describe-stacks --region us-west-2 \
  --stack-name PiiDocumentRedactionPocStack \
  --query 'Stacks[0].StackStatus' --output text      # UPDATE_COMPLETE / CREATE_COMPLETE
```

### 1.5 Find the API endpoint and sign-in settings

The stack outputs the API endpoint and the Cognito user pool details, which the frontend configuration needs:

```bash
aws cloudformation describe-stacks --region us-west-2 \
  --stack-name PiiDocumentRedactionPocStack \
  --query "Stacks[0].Outputs[].[OutputKey,OutputValue]" --output text
# ApiUrl            https://xxxxxxxx.execute-api.us-west-2.amazonaws.com
# UserPoolId        us-west-2_XXXXXXXXX
# UserPoolClientId  xxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 1.5.1 Roles and user management

The system has two roles:

| Role | What it can do |
| --- | --- |
| Administrator (Cognito `admins` group) | Upload and process documents; on the "User management" page invite administrators or regular users, change roles, resend invitations and delete users; on the "Rule management" page maintain redaction rules |
| Regular user | Can only upload documents and process and download their own jobs. Cannot see rules or user management |

Typical flow:

1. Specify the first administrator with `-c adminEmail=...` at deploy time. That person receives an invitation email.
2. The administrator signs in to the frontend with the email and temporary password, then sets a new password when prompted (at least 12 characters, with upper and lower case letters and digits).
3. On the "User management" page enter an email, choose a role and click "Send invitation". The temporary password is valid for 3 days; after that use "Resend invitation".

Rules and limits:

- **Accounts exist only by invitation:** self sign-up is disabled on the user pool.
- **You cannot lock yourself out:** an administrator cannot demote or delete themselves, and the last administrator cannot be removed.
- **When role changes take effect:** demoting or deleting a user revokes their sign-in sessions, but access tokens already issued stay valid until they expire (up to 1 hour). Promotion to administrator takes effect after that user signs in again.
- **Jobs are isolated per user:** every job records its uploader (`ownerId`). Other users' jobs are always answered with 404 as if they do not exist, and administrators cannot see other users' documents either.
- **Every API requires sign-in:** a request without a token gets 401, and a regular user calling an admin endpoint gets 403. The API as a whole is throttled to 10 requests per second with a burst of 20 (the `API_THROTTLE_*` constants).
- **Custom regex:** rules are maintained by trusted administrators. Custom regex patterns have no execution timeout, so a badly written pattern (for example, nested repetition) can slow processing down or make a job time out.
- **Email volume:** invitation emails are sent by the default Cognito sender, which allows at most 50 emails per day. With many users, configure Amazon SES on the user pool.

If no administrator can sign in (for example the only administrator has left), redeploy with a new email using `-c adminEmail=new-admin@example.com`, or operate on the user pool directly with the AWS CLI:

```bash
aws cognito-idp admin-add-user-to-group --region us-west-2 --user-pool-id <UserPoolId> \
  --username someone@example.com --group-name admins
```

### 1.6 CORS: allowed frontend origins

The API's CORS allow-list is `cors_preflight.allow_origins` in `backend/infra/redaction_stack.py` (it includes `http://localhost:5173` by default). **If you host the frontend on another origin, add it there and run `cdk deploy` again, otherwise browsers will fail with CORS errors.**

### 1.7 End-to-end check

```bash
BASE=https://<api-id>.execute-api.us-west-2.amazonaws.com
# Get an access token (the user must already have set a permanent password)
TOKEN=$(aws cognito-idp initiate-auth --region us-west-2 \
  --client-id <UserPoolClientId> --auth-flow USER_PASSWORD_AUTH \
  --auth-parameters USERNAME=user@example.com,PASSWORD='<password>' \
  --query AuthenticationResult.AccessToken --output text)
AUTH="Authorization: Bearer $TOKEN"

# Upload (form field "file", .pdf or .docx, at most 4 MB, content must match the extension); returns {"jobId": ..., "status": "processing", "mode": ..., "fileName": ...}
curl -s -H "$AUTH" -X POST "$BASE/jobs" -F "file=@sample.docx;filename=sample.docx"

# Poll until status becomes "completed" (or "failed")
curl -s -H "$AUTH" "$BASE/jobs/<jobId>"

# Download the redacted file
curl -s -H "$AUTH" -o redacted.docx "$BASE/jobs/<jobId>/download"
```

To confirm the agent runs in its Strands form, search CloudWatch Logs:

```bash
aws logs describe-log-groups --region us-west-2 \
  --log-group-name-prefix /aws/bedrock-agentcore/runtimes/pii_redaction_agent
# In the matching log group search for: "Redaction Strands agent (12 registered tools) handling invocation"
```

### Processing time

Documents with many embedded images are slower: each image needs one multimodal OCR call (roughly 8 to 17 seconds), and each text paragraph needs one LLM call and one Guardrails call. These calls run **concurrently**; tune the concurrency with environment variables on the agent runtime:

| Environment variable | Default | Purpose |
| --- | --- | --- |
| `OCR_MAX_WORKERS` | 4 | Concurrent image OCR calls |
| `LLM_MAX_WORKERS` | 8 | Concurrent per-paragraph LLM PII detection calls |
| `GUARDRAILS_MAX_WORKERS` | 8 | Concurrent per-paragraph Guardrails detection calls |

For reference, a 1.3 MB `.docx` with 17 images and about 576 paragraphs completes in roughly 2 minutes with concurrency. Raising concurrency can trigger Bedrock throttling (calls have built-in bounded retries).

### Detection scope and false-positive control

Only information about **natural persons** counts as PII: names, contact details, addresses specific to a street or building, dates of birth and ages, ID numbers, bank card and account numbers, credentials, license plates and similar. Company, organization, product and cloud service names, bare country or city names, availability zones, website URLs and hardware specifications are **not** PII. If a customer name must also be hidden, add a keyword rule in the frontend's rule management page instead of relying on the model.

False positives are controlled by several layers:

| Layer | Location | What it does |
| --- | --- | --- |
| LLM prompt v3 | `PROMPT_TEMPLATE` in `agent/tools/llm_pii_detect.py` | States that PII means natural persons only and lists categories that are not PII |
| Name and address source defaults to "LLM only" | `DETECTION_SOURCE_DEFAULT` in `shared/models.py` | In Chinese business documents Guardrails often labels company and product names as NAME and place names as ADDRESS. Users can still choose "both" at upload time |
| Format validation | `agent/tools/span_validation.py` | Drops model results with the wrong format: URLs are never reported; MAC, IP, email, phone and card numbers must match their format; a bare number is not an AGE unless next to an age marker; overlong spans are dropped. User-defined regex rules are not affected |
| Review list merged by text | `agent/tools/detection_results.py` | Each distinct value appears once, listing all its sources and types; selecting or clearing a value applies to all its occurrences |

After adjusting these rules, test once with a sample document that contains real-looking PII and once with a technical document that contains none, to confirm recall did not drop and false positives did not return.

## 2. Deploy the frontend

Run all commands from the `frontend/` directory.

```bash
cd frontend
npm ci
```

### Local development

Put the backend address and the Cognito settings in `frontend/.env.development.local` (gitignored), then start the dev server. Use `.env.development.local`: in development mode `.env.development` takes precedence over `.env.local`.

```bash
cat > .env.development.local <<'ENV'
VITE_API_BASE_URL=https://<api-id>.execute-api.us-west-2.amazonaws.com
VITE_COGNITO_REGION=us-west-2
VITE_COGNITO_CLIENT_ID=<UserPoolClientId>
ENV
npm run dev            # http://localhost:5173 by default
```

The URL must be `https://`; a non-HTTPS value is rejected when the API client is constructed. If `VITE_COGNITO_REGION` or `VITE_COGNITO_CLIENT_ID` is missing, the app fails at startup. A production bundle built without `VITE_API_BASE_URL` throws as soon as the app loads, on purpose (there is no localhost fallback). The build command itself still succeeds, so set the variable at build time. **Do not commit a real API URL into any `.env.*` file.**

### Production build

```bash
VITE_API_BASE_URL=https://<api-id>.execute-api.us-west-2.amazonaws.com \
VITE_COGNITO_REGION=us-west-2 VITE_COGNITO_CLIENT_ID=<UserPoolClientId> npm run build
```

The output in `frontend/dist/` is plain static files that can be hosted anywhere (S3 with CloudFront, Nginx, and so on). The hosting origin must be added to the backend CORS allow-list (see section 1.6).

### Verify

```bash
npm run typecheck
npm run lint
npm test
```

### UI language (Chinese / English)

The dropdown on the right of the top bar switches the UI language. The choice is stored in browser localStorage (key `pii-redaction.lang`); the default is Simplified Chinese. Switching also updates the page title and `<html lang>`.

- All UI strings live in the dictionary in `frontend/src/i18n/index.tsx`. Each key has both a Chinese and an English entry. Components use `const { t } = useI18n()` and call `t('key')`; variables use `{name}` placeholders, for example `t('review.selectRow', { value })`. Keys are type-checked, so a typo fails compilation.
- To add a string, add a `{ zh, en }` entry; the `{placeholders}` must match in both languages. To add a language, add a value to `Lang`, an item to `LANGUAGES`, and fill in every dictionary entry.
- Not translated: file names, detected original text and its context, type identifiers in Debug mode, and regular expressions.
- Job failure reasons: the backend does not store user-facing sentences. It stores an `errorCode` (plus optional `errorParams`, such as the failing OCR page `{"page": 3}`), returned by `GET /jobs/{jobId}`, and the frontend translates it through `job.error.<CODE>`. `errorMessage` keeps an English diagnostic for troubleshooting and is **not** shown to users. Existing codes are in `backend/agent/tools/job_status_update.py` (`REVIEW_TIMEOUT`, `STRUCTURE_VERIFICATION_FAILED`, `OCR_PAGE_FAILED`, `OCR_FAILED`, `PROCESSING_FAILED`) and `upload_handler.py` (`STORAGE_WRITE_FAILED`). To add a failure reason, call `update_job_status` with a new `error_code` in the backend and add a `job.error.<CODE>` entry to `frontend/src/i18n/index.tsx`. Unknown codes, or older jobs without a code, fall back to `errorMessage`.
- Without a `LanguageProvider` (for example when rendering a component alone in a test) the UI renders in Chinese.

### Frontend timeouts

- Single API request timeout: 30 seconds (`DEFAULT_TIMEOUT_MS` in `src/api/client.ts`).
- Processing timeout: if a job stays in `processing` for more than **20 minutes**, the UI tells the user to go back and retry (`PROCESSING_TIMEOUT_MS` in `src/views/UploadView.tsx`).

## Cleanup

Delete the stack from the `backend/` directory:

```bash
cdk destroy -c adminEmail=admin@example.com
```

`cdk destroy` does not remove the following; handle them yourself if needed:

- A few empty Lambda log groups may remain (see "Leftover log groups" in [`docs/least-privilege-deploy-policy.md`](docs/least-privilege-deploy-policy.md)). They contain no document content and can be deleted manually.
- The `CDKToolkit` stack created by `cdk bootstrap` (`CDKToolkit-PiiRedaction` with a separate qualifier) and the CDK assets bucket. Other CDK applications in the account may use them, so check before deleting.
- The three managed policies created as described in `docs/` (`PiiRedactionCdkExecution`, `PiiRedactionCdkExecutionIam`, `PiiRedactionAppBoundary`), the deploy identity and the bootstrap policy must be deleted manually.
- Records in Bedrock model invocation logging (if the account has it on) are outside what this application manages; see "Data residency and compliance".

## Troubleshooting

**`ResolutionImpossible` / pywin32 while building the bundle:** you are on Windows. `strands-agents` depends on `mcp`, which requires `pywin32; sys_platform == "win32"`, and pip tries to resolve a pywin32 that has no aarch64 wheel. Build on a Linux host (or WSL).

**Resources were created in the wrong region:** see Prerequisites. Run `cdk destroy` in the wrong region, then deploy again in us-west-2.

**A job stays in `processing` and the agent logs never mention its jobId:** the job was not dispatched to the agent. The `JobDispatcher` Lambda reads new jobs from the DynamoDB stream and calls `InvokeAgentRuntime`, which only returns when the agent entrypoint returns. So the entrypoint in `agent/main.py` must return immediately and run the pipeline in a background thread (keeping the session alive with `app.add_async_task`). If the entrypoint becomes synchronous, a review-mode job makes the Lambda wait out its 300-second timeout, the stream retries the whole batch, and every job uploaded afterwards queues up behind it. When investigating, check `Duration` in the `JobDispatcher` logs; it should be a few seconds. `handle_job` only processes jobs whose status is `processing`, so duplicate dispatches are skipped. The event source mapping reads the stream from `TRIM_HORIZON`, so jobs submitted right after a fresh deployment are dispatched too, once the mapping starts working (usually within a few tens of seconds).

**The frontend says the processing timed out:** the job really is slow, or the backend is stuck. Check the status with `GET /jobs/{jobId}`, then the agent's CloudWatch logs.

**CORS errors in the browser:** the frontend origin is not in `allow_origins`; see section 1.6.

**Changes under `agent/` or `shared/` do not show up in the deployed behavior:** the bundle was not rebuilt. Run `scripts/build_agent_bundle.py`, then `cdk deploy`.

## Design constraints (read before changing the code)

1. `handle_job` must stay deterministic. Do not change it to let the Strands agent's reasoning loop call the tools. The Strands agent is decorative at runtime on purpose.
2. `agent/tools/llm_pii_detect.py` must not import strands (`test_2_16` enforces this). Strands code belongs only in `agent/redaction_agent.py` and the `invoke` entrypoint in `main.py`.
3. `process_invocation` in `agent/runtime.py` imports neither strands nor bedrock_agentcore.
4. LLM detection sources may be added but never removed, and must never raise.
5. The deployment region is fixed to `us-west-2`.

## Key constants

- `LLM_PII_MODEL_ID = us.anthropic.claude-haiku-4-5-20251001-v1:0` (sends only `temperature=0.0`, not `topP`)
- `IMAGE_OCR_MODEL_ID = us.anthropic.claude-sonnet-4-5-20250929-v1:0`
- Both are defined in `backend/infra/redaction_stack.py`.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This library is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file. Third-party components keep their own licenses; see [THIRD-PARTY-LICENSES](THIRD-PARTY-LICENSES).
