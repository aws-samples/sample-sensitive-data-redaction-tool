# Least-privilege deployment permissions

This directory contains five IAM policies, in three groups by purpose:

| File | Managed policy name (suggested) | Attach to | Purpose |
| --- | --- | --- | --- |
| `least-privilege-deploy-policy.json` | Your choice | The **deployment identity** (IAM user or role) that runs `cdk deploy` and `cdk diff`, attached permanently | Can only assume the CDK deploy, file-publishing and lookup roles, read this application's stack, and run admin recovery on this application's user pool. Cannot create resources or change IAM |
| `least-privilege-bootstrap-policy.json` | Your choice | The deployment identity, **attached only while running `cdk bootstrap`** and detached right after | Creates the `CDKToolkit` stack, the CDK asset bucket, the bootstrap roles and the version parameter. It can change the bootstrap roles, so it must not stay attached; attach it again only for a later re-bootstrap (for example, a CDK upgrade) |
| `least-privilege-cfn-exec-policy.json` | `PiiRedactionCdkExecution` | The CDK **CloudFormation execution role** (separate qualifier, optional hardening) | Replaces the default `AdministratorAccess` of bootstrap, and only allows creating, updating and deleting this application's resources (other than IAM) |
| `least-privilege-cfn-exec-iam-policy.json` | `PiiRedactionCdkExecutionIam` | Same as above, attached to the execution role together with the previous one | IAM role operations for this application. The execution role can only create and modify roles that carry the `PiiRedactionAppBoundary` permissions boundary |
| `least-privilege-app-boundary-policy.json` | `PiiRedactionAppBoundary` | Not attached directly: used as the **permissions boundary of every IAM role the stack creates** | The most that application roles (Lambda, AgentCore Runtime, custom resources) can do. Contains no `iam:*` or `sts:*` |

The execution policy is split in two because, with the boundary conditions added, a single policy would exceed the 6144-character limit for managed policies. The first two files are the deployment identity's permissions; the last three control what CloudFormation can create and what the created roles can do. Each role in the stack still has its own policy scoped to specific resources; the boundary is only their upper limit.

## Basis

The policies were derived from the CloudTrail records of actual deployments in a test account (several `cdk deploy` runs on 2026-11-03, including updates that added Cognito, the authorizer and the user management endpoints).

CloudTrail shows that when deploying with a modern CDK bootstrap:

- **The deployment identity itself** only called `sts:AssumeRole` (assuming `cdk-hnb659fds-deploy-role`, `file-publishing-role` and `lookup-role`) and a few read-only queries.
- **All resources were created by the CloudFormation execution role** (`cdk-hnb659fds-cfn-exec-role`): Lambda, S3, DynamoDB, SQS, API Gateway, Cognito, the Bedrock Guardrail, the AgentCore Runtime and IAM roles.
- This application has no Docker image assets, so it needs neither the `image-publishing-role` nor ECR push permissions.

All five policies passed IAM Access Analyzer `validate-policy` with 0 findings, and all are within the 6144-character limit for managed policies when minified (1120, 1561, 5045, 1475 and 2447 characters, in the order of the table). The escalation paths were also checked with the IAM policy simulator: the deployment identity cannot modify the CDK roles or the execution policies; the execution role cannot create a role without the boundary, cannot attach `AdministratorAccess` and cannot remove the boundary; the boundary allows no IAM actions and no access to buckets, tables or user pools outside this application.

## Daily deploy policy (`least-privilege-deploy-policy.json`)

| Sid | Purpose |
| --- | --- |
| `CdkAssumeDeployRoles` | Assume the CDK deploy, file-publishing and lookup roles. `cdk deploy` and `cdk diff` need only this statement |
| `AppStackRead` | Read this application's stack status and outputs (step 1.5 of the README, to get the API URL and user pool ID) |
| `ApiEndpointLookup` | `aws apigatewayv2 get-apis`. It can only list APIs, not modify them |
| `AdminRecoveryOnUserPool` | For the README case "no administrator can sign in", to add someone back to the `admins` group from the CLI. It applies only to user pools tagged `PiiRedactionApp=true` (the stack tags this application's user pool). Delete it if you do not need it |
| `AccountWideUnscopable` | `sts:GetCallerIdentity`, `cloudformation:ValidateTemplate` and `GetTemplateSummary`, which do not support resource-level scoping |

The deployment identity itself has **no** permission to write Lambda functions, S3 data, DynamoDB or IAM, and it cannot call `iam:PassRole` (the deploy role hands the execution role to CloudFormation).

## Bootstrap policy (`least-privilege-bootstrap-policy.json`)

| Sid | Purpose |
| --- | --- |
| `BootstrapToolkitStack` | Create and update the `CDKToolkit` stack |
| `BootstrapAssetBucket` / `BootstrapContainerRepo` | The CDK asset bucket `cdk-hnb659fds-*` and container repository |
| `BootstrapRoles` | Create and configure the `cdk-hnb659fds-*` roles |
| `BootstrapExecutionPolicy` | Read and update the three managed policies `PiiRedactionCdkExecution`, `PiiRedactionCdkExecutionIam` and `PiiRedactionAppBoundary` |
| `BootstrapVersionParameter` | The SSM parameter `/cdk-bootstrap/*/version` |
| `AccountWideUnscopable` | Same as above |

`BootstrapRoles` allows attaching policies to the `cdk-hnb659fds-*` roles, and `BootstrapExecutionPolicy` allows changing the execution policies. Both can be used to escalate privileges (for example, by attaching `AdministratorAccess` to the deploy role and then assuming it). So attach this policy **only while running `cdk bootstrap`, and detach it right after**; attach it again only for a later re-bootstrap (for example, a CDK upgrade).

`BootstrapToolkitStack`, `BootstrapAssetBucket` and `BootstrapContainerRepo` use the service-wide wildcards `cloudformation:*`, `s3:*` and `ecr:*`, with resources scoped to CDK bootstrap's own stack, bucket and repository. This policy is attached only during bootstrap; before using it in production, narrow these to the actions bootstrap actually calls, based on CloudTrail (see "What is not verified").

## Important: the actual permission boundary under the default bootstrap

A default `cdk bootstrap` attaches **`AdministratorAccess`** to the CloudFormation execution role, and that role is shared by every CDK application in the same account and Region. Therefore:

- The deployment identity policy is small, but **anyone who can assume the deploy role can deploy arbitrary resources through CloudFormation**.
- You cannot simply replace the shared execution role with this application's narrowed policy, because other CDK applications in the account would fail to deploy.

Real least privilege requires a **separate bootstrap for this application** (its own qualifier), with the two execution policies attached to its execution role and the permissions boundary on every role in the stack:

```bash
Q=piired                                     # your choice, at most 10 lowercase letters or digits
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

# 0. Replace hnb659fds in all policies with your qualifier, and CDKToolkit in the bootstrap policy with the toolkit stack name used below
sed -i "s/hnb659fds/$Q/g" docs/least-privilege-*.json
sed -i "s#stack/CDKToolkit/#stack/CDKToolkit-PiiRedaction/#" docs/least-privilege-bootstrap-policy.json

# 1. Create the three managed policies (run by the account administrator)
aws iam create-policy --policy-name PiiRedactionCdkExecution \
  --policy-document file://docs/least-privilege-cfn-exec-policy.json
aws iam create-policy --policy-name PiiRedactionCdkExecutionIam \
  --policy-document file://docs/least-privilege-cfn-exec-iam-policy.json
aws iam create-policy --policy-name PiiRedactionAppBoundary \
  --policy-document file://docs/least-privilege-app-boundary-policy.json

# 2. Attach the bootstrap policy to the deployment identity temporarily (the daily deploy policy stays attached)
aws iam create-policy --policy-name pii-redaction-bootstrap \
  --policy-document file://docs/least-privilege-bootstrap-policy.json
aws iam attach-role-policy --role-name <YOUR_DEPLOY_ROLE> \
  --policy-arn arn:aws:iam::$ACCOUNT:policy/pii-redaction-bootstrap

# 3. Bootstrap with the deployment identity, passing both execution policies separated by a comma
cdk bootstrap aws://$ACCOUNT/us-west-2 \
  --qualifier $Q --toolkit-stack-name CDKToolkit-PiiRedaction \
  --cloudformation-execution-policies "arn:aws:iam::$ACCOUNT:policy/PiiRedactionCdkExecution,arn:aws:iam::$ACCOUNT:policy/PiiRedactionCdkExecutionIam"

# 4. Detach the bootstrap policy right away
aws iam detach-role-policy --role-name <YOUR_DEPLOY_ROLE> \
  --policy-arn arn:aws:iam::$ACCOUNT:policy/pii-redaction-bootstrap

# 5. Add two entries to "context" in backend/cdk.json: use this qualifier, and put the permissions boundary on every IAM role in the stack
#    "@aws-cdk/core:bootstrapQualifier": "piired",
#    "@aws-cdk/core:permissionsBoundary": { "name": "PiiRedactionAppBoundary" }
```

`@aws-cdk/core:permissionsBoundary` in step 5 is a built-in CDK context. It adds `PermissionsBoundary` to every IAM role in the stack, including the custom resource roles that aws-cdk-lib adds. Add it only on this least-privilege path: an account with the default bootstrap has no `PiiRedactionAppBoundary` policy, and the deployment would fail. After that, the deployment identity only needs the edited `least-privilege-deploy-policy.json` permanently.

**Stacks that are already deployed:** if the stack was deployed before the permissions boundary was added (with the default bootstrap, or with an earlier version of the execution policy in this document), its roles do not carry `PiiRedactionAppBoundary`. The new execution policy only allows modifying and deleting roles that carry this boundary, so first run `cdk destroy` with the original execution role, then switch to the policies in this document and deploy again. Otherwise the delete stops in `DELETE_FAILED` because `iam:DeleteRole` is denied.

## Execution role policies (`least-privilege-cfn-exec-policy.json` + `least-privilege-cfn-exec-iam-policy.json`)

Resources are scoped by this application's naming:

- **S3:** buckets `piidocumentredactionpocst*` (the truncated stack-name prefix); read-only access to the CDK asset bucket, because creating a Lambda function and an AgentCore Runtime requires reading the code package. `AppBuckets` has the condition `aws:ResourceAccount` = `${aws:PrincipalAccount}`: S3 bucket names are globally unique and this prefix is public and predictable, so another account could register a bucket that matches it first. The condition makes sure the execution role can only act on buckets in its own account, which prevents bucket-name squatting. The CDK asset bucket name already contains the account ID, so `ReadCdkAssets` does not need the condition.
- **Lambda:** functions `PiiDocumentRedactionPocSt*` and event source mappings.
- **DynamoDB:** tables `RedactionJobs` (including its stream), `RedactionRules` and `RuleOperations`.
- **SQS:** queue `RuleOperationsQueue.fifo`.
- **IAM** (in `least-privilege-cfn-exec-iam-policy.json`): roles `PiiDocumentRedactionPocSt*`.
  - `AppRoles`: creating, deleting and modifying roles and inline policies, and setting the boundary, all require `iam:PermissionsBoundary` to be `PiiRedactionAppBoundary`. A role without this boundary can be neither created nor modified.
  - `AppRolesReadAndTag`: reading and tagging, with no condition (these actions do not support that condition key and cannot change permissions).
  - `AttachRolePolicy` may only attach `AWSLambdaBasicExecutionRole`, and the target role must carry the same boundary.
  - `DeleteRolePermissionsBoundary` is not granted, and the `PiiRedaction*` managed policies themselves cannot be modified, so the execution role cannot remove or widen the boundary.
  - `PassRole` may only pass to `lambda.amazonaws.com` and `bedrock-agentcore.amazonaws.com`.
  - Creating a service-linked role is allowed only for `bedrock-agentcore`, which is needed the first time a Runtime is created in a new account.
- **AgentCore:** Runtime `pii_redaction_agent-*` and its workload identity.
- **KMS:** usable only through S3, Lambda, SQS and DynamoDB (the `kms:ViaService` condition), for each service's AWS managed key.
- **CloudWatch Logs:** `LogGroups` only allows creating, deleting and configuring this application's log groups `/aws/*/PiiDocumentRedactionPoc*`, that is, the API access log group `/aws/apigateway/PiiDocumentRedactionPocApi-access` and the Lambda log groups `/aws/lambda/PiiDocumentRedactionPocSt*`. The stack creates these log groups itself with a 1-day retention. AgentCore creates the AgentCore Runtime log group itself with the runtime's execution role when the runtime is created, not through the CloudFormation execution role; the stack sets its retention, and deletes it with the stack, through custom resources that can act only on that one log group. `ApiLogDelivery` holds the log delivery permissions that API Gateway requires to turn on access logging (`*LogDelivery`, `PutResourcePolicy` and others); these actions do not support resource-level scoping and can only use `"Resource": "*"`.

These three groups keep service-wide wildcard actions, but the resources are scoped. Before using them in production, narrow them to the actions CloudFormation actually calls (see "What is not verified"):

| Sid | Actions | Resource | Reason |
| --- | --- | --- | --- |
| `AppBuckets` | `s3:*` | This application's two buckets | CloudFormation uses many actions to read and write bucket configuration (about 30 in CloudTrail), and deleting the stack also empties and deletes the buckets |
| `AppFunctions` | `lambda:*` | This application's functions | Create, update, delete and invoke custom resources; the actions change between versions |
| `HttpApi` / `UserPool` | `apigateway:*` / 17 explicit `cognito-idp:*` actions | `/apis*`, `userpool/*` | API Gateway authorizes per HTTP method. Cognito user pool IDs are random and cannot be scoped by name, but the actions are listed one by one (create, update, delete and read for the pool, client and groups, plus tagging) and include no operation on user data |

These actions do not support resource-level scoping and can only use `"Resource": "*"`: `bedrock:CreateGuardrail`, `bedrock-agentcore:CreateAgentRuntime`, `cognito-idp:CreateUserPool`, `lambda:GetAccountSettings`, `iam:GetAccountSummary`, and the Lambda event source mapping actions `Create/Get/Update/Delete/ListEventSourceMapping`. The last group was added after testing: when a mapping no longer exists (CloudFormation's confirmation check after deleting it, or a retry after a failed delete), Lambda authorizes the call against resource `*` and the request carries no function ARN. So both scoping the resource to `event-source-mapping:*` and adding a `lambda:FunctionArn` condition are denied, and the stack gets stuck in DELETE_FAILED (tested twice on 2026-11-11). The trade-off is that this execution role can modify and delete any event source mapping in the account; it is used only by CloudFormation, and with the separate bootstrap it serves only this application.

`AgentRuntimeCreateTimeChecks` was also added after testing. `CreateAgentRuntime` creates the DEFAULT endpoint and tags it at the same time, when the Runtime ID has not been generated yet. AgentCore then checks `CreateAgentRuntimeEndpoint` and `TagResource` against the literal `runtime/*`, which the name-scoped `runtime/pii_redaction_agent-*` does not match, so it has to be opened up to `runtime/*`. Deleting a Runtime also needs `DeleteAgentRuntimeEndpoint`, which can be scoped by name and is in the `AgentRuntime` statement. None of this is visible in CloudTrail or in error messages (CloudFormation only reports `Access denied for operation 'AWS::BedrockAgentCore::Runtime'`); it was found by calling the APIs directly with the same policy.

## Permissions boundary (`least-privilege-app-boundary-policy.json`)

`PiiRedactionAppBoundary` is the upper limit of every IAM role in the stack. What a role can actually do is the intersection of its own policies and the boundary; even if the template gave a role broader permissions, it could not go beyond the boundary. The boundary scopes resources by this application's naming:

| Sid | Allows |
| --- | --- |
| `AppBuckets` / `ReadCdkAssets` | This application's two buckets `piidocumentredactionpocst*` (`s3:*`, including what the S3 auto-delete function gets through the bucket policies, and only for buckets in the same account, with the same condition as the execution policy's `AppBuckets`); read-only access to the CDK asset bucket |
| `AppTables` / `ListStreams` | Tables `RedactionJobs` (including its stream), `RedactionRules` and `RuleOperations`; `dynamodb:ListStreams` does not support resource-level scoping |
| `AppQueue` | Queue `RuleOperationsQueue.fifo` |
| `AppFunctions` | Invoking and reading this application's functions (the custom resource framework functions invoke the handlers) |
| `AppLogGroups` / `DescribeLogGroups` | This application's Lambda log groups and the AgentCore Runtime log group (including log writes from `AWSLambdaBasicExecutionRole`) |
| `AppUserPool` | Cognito admin APIs, only on user pools tagged `PiiRedactionApp=true` |
| `BedrockModels` / `Guardrails` | Invoking models and inference profiles, and applying the Guardrail |
| `AgentRuntime` / `AgentCoreMetrics` | Invoking this application's Runtime, getting workload access tokens, and writing metrics in the `bedrock-agentcore` namespace |
| `NoResourceLevelPermissions` | `textract:DetectDocumentText` and X-Ray writes, which do not support resource-level scoping |

`s3:*`, `dynamodb:*`, `sqs:*` and `logs:*` are the boundary's upper limit, with resources scoped to this application. The boundary itself grants no permissions; what a role can actually do is decided by its own policies. Before using it in production, you can narrow these to specific actions as needed (see "What is not verified").

The boundary allows no `iam:*` or `sts:*`, so application roles cannot create or modify roles and cannot assume other roles. When a resource policy (for example, a bucket policy) grants permissions to a role ARN, the boundary limits those as well.

## Test results (2026-11-10, us-west-2)

- A new role, `PiiRedactionDeployer`, had only `least-privilege-deploy-policy.json` attached (with the qualifier changed to `piired`; at that time this policy still included the bootstrap permissions, which were split out later, see the 2026-11-16 record below). It completed every step:
  - `cdk bootstrap --qualifier piired --toolkit-stack-name CDKToolkit-PiiRedaction`, including creating the `PiiRedactionCdkExecution` policy;
  - the first `cdk deploy` failed for lack of the AgentCore endpoint permissions above, and CloudFormation **rolled back completely** with the narrowed execution role, deleting about 100 resources it had created;
  - after adding the permissions, the redeployment succeeded: CDK first deleted the stack in `ROLLBACK_COMPLETE` and then created it again, with all 119 resources succeeding.
- Verification after deployment: the first administrator received the invitation (`FORCE_CHANGE_PASSWORD`, in the `admins` group); a request without a token got 401, a regular user calling `/rules` or `/users` got 403, and an administrator got 200.

### Follow-up checks (same day, same `piired` environment)

- **Cognito action list:** it was previously assumed that CloudTrail had no Cognito events, which was wrong. Querying by the deployment time window found 7 actions on the create path, and `cognito-idp:*` was replaced with an explicit list. After that, three creates, one update and three deletes all succeeded.
- **Update with changes:** temporarily changed the API stage throttling, the user pool (temporary password validity), the user pool client (refresh token validity), the lifecycle of both S3 buckets and one Lambda environment variable. The deployment succeeded, and reverting the values and deploying again also succeeded.
- **`cdk destroy`:** run with the least-privilege identity after putting a placeholder file in each of the two buckets. The buckets were emptied and deleted automatically, and nothing was left behind: tables, queue, Lambda functions, event source mappings, user pool, AgentCore Runtime, Guardrail, IAM roles and API. The first destroy failed for lack of Get/Delete permissions on event source mappings, and succeeded after they were added.
- **Leftover log groups (known, harmless):** when the stack is deleted, the 6 Lambda functions behind the custom resources are invoked one last time. Lambda delivers logs asynchronously and recreates the log group after it was deleted, with no retention period. Changing the deletion order did not help (the event timeline shows the correct order and the log group is still recreated). The logs of these 6 functions contain only bucket names, log group names and CloudFormation request information, no document content or user data, so no handling logic was added. After a destroy a few empty `/aws/lambda/PiiDocumentRedactionPocSt-*` log groups with no retention remain and can be deleted manually.
- **Trying a condition on event source mappings** (2026-11-11): after adding a `lambda:FunctionArn` condition to the event source mapping Get/Update/Delete actions, `cdk destroy` first failed because the confirmation check `GetEventSourceMapping` after the delete was denied. With Get made unconditional, the retried delete then failed because `DeleteEventSourceMapping` (on a mapping that no longer existed) was denied. The permissions were therefore restored to unconditional; after that, the retried delete, a fresh deploy, `cdk destroy` and another fresh deploy all succeeded.

### Split policies and permissions boundary (2026-11-16, same `piired` environment)

- The existing stack was first removed with `cdk destroy` under the old execution role (see "Stacks that are already deployed").
- The account administrator created `PiiRedactionCdkExecution` (new version), `PiiRedactionCdkExecutionIam` and `PiiRedactionAppBoundary`. The deployment identity ran `cdk bootstrap` with the bootstrap policy attached temporarily; once the execution role had both execution policies, the bootstrap policy was detached and the deployment identity kept only the daily deploy policy.
- A fresh `cdk deploy` succeeded on the first try. All 19 IAM roles carry `PiiRedactionAppBoundary`, and the user pool carries the `PiiRedactionApp=true` tag.
- Functionality: an end-to-end redaction job completed; inviting, promoting, demoting and deleting users, and creating and deleting rules, all succeeded; Lambda can still decrypt its environment variables under the boundary.
- The deployment identity recovered an administrator with `admin-add-user-to-group`, which succeeded on this application's user pool.
- Escalation attempts were all denied: the deployment identity attaching `AdministratorAccess` to `cdk-piired-deploy-role` (`iam:AttachRolePolicy`) and creating a new version of `PiiRedactionCdkExecution` (`iam:CreatePolicyVersion`). A temporary stack, deployed through the execution role, that creates a role without the boundary under this application's prefix was denied `iam:CreateRole`; the stack rolled back and the role was never created.
- A `cdk deploy` with no code changes reported no changes; `cdk destroy` succeeded, leaving only the empty Lambda log groups described in "Leftover log groups"; another fresh deploy succeeded.

## What is not verified

- **IAM Access Analyzer policy generation:** the checks above only cover the actions this application actually uses. Before going to production, use [policy generation](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-policy-generation.html) to narrow the service-wide wildcards in three policies to explicit action lists based on CloudTrail: `s3:*`, `lambda:*` and `apigateway:*` in the execution policy; `cloudformation:*`, `s3:*` and `ecr:*` in the bootstrap policy; and `s3:*`, `dynamodb:*`, `sqs:*` and `logs:*` in the boundary policy.
- **Rollback path:** the first deployment failure was rolled back once by CloudFormation with the narrowed role, and the delete path is covered by destroy. Rolling back to the previous version after a failed update (UPDATE_ROLLBACK) was not tested separately.
- **Account and Region:** the account and Region in the policies are `*`. If you only deploy to `us-west-2`, you can replace `*:*` in the ARNs with `us-west-2:<account ID>` to narrow them further.

## References

- [AWS CDK bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping.html)
- [Customize AWS CDK bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping-customizing.html)
- [Create and apply permissions boundaries for the AWS CDK](https://docs.aws.amazon.com/cdk/v2/guide/customize-permissions-boundaries.html)
