# Architecture deep-dive

A walkthrough of how the system is wired together, what each module owns, and the failure modes each was designed for. Covers more ground than the [README](../README.md) — written for readers who want the full picture without seeing the actual source.

## Boundaries at a glance

```
┌─────────────────────────────────────────────────────────────────────┐
│                          Public surfaces                            │
│  explainmyscanapp.com (marketing)     dev.explainmyscanapp.com (app)│
│  www.explainmyscanapp.com (alias)     api.dev.explainmyscanapp.com  │
│                                       auth.dev.explainmyscanapp.com │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       CloudFront / API Gateway                      │
│  TLS termination, WAF, request validation, Cognito JWT authorizer   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
│   Auth lambdas   │   │  Sync API λs     │   │  Async pipeline  │
│  (Cognito IDP    │   │  upload / read   │   │  process / dlq / │
│   direct calls)  │   │  / export        │   │  textract callback│
└──────────────────┘   └──────────────────┘   └──────────────────┘
                                  │                   │
                                  ▼                   ▼
                          ┌──────────────────────────────────┐
                          │   S3 + DynamoDB (KMS encrypted)  │
                          │   SQS + SNS (KMS encrypted)      │
                          │   Bedrock + Textract (BAA)       │
                          │   SES + CloudWatch (BAA)         │
                          └──────────────────────────────────┘
```

## Module-by-module

### `auth` — Cognito + custom domain
- **User Pool**: email username, MFA-required TOTP, 12-char password policy with mixed case + digits + symbols, `verify-before-update` on email so a typo doesn't lock anyone out.
- **App client** with PKCE-only flow, scopes `openid email profile aws.cognito.signin.user.admin` (the last one is the non-obvious requirement for SPA-side ChangePassword / DeleteUser / UpdateUserAttributes).
- **Custom domain** at `auth.dev.explainmyscanapp.com` backed by us-east-1 ACM cert + Route 53 A alias to Cognito's CloudFront distribution (zone ID `Z2FDTNDATAQYW2`). Cognito requires the parent domain to have an A record first; nesting the subdomain under `dev.<root>` reuses the app's frontend alias and dodges apex-DNS gymnastics.
- **Hosted UI customization**: brand cyan submit button, focus ring, redirect links, 224-px brand logo (Cognito caps `imageFile` at ~96 KB raw / 128 KB base64).

### `frontend` — SPA via S3 + CloudFront
- KMS-encrypted bucket, blocked public access, CloudFront with **OAC** (not legacy OAI) so the bucket policy can require `AWS:SourceArn` matches the distribution.
- **Security headers policy**: HSTS preload, CSP allow-listing only the origins we actually call (API custom domain, Cognito IDP, Cognito hosted UI, RUM dataplane, Cognito Identity, STS), frame-ancestors `none`.
- SPA fallback via 403/404 custom error responses → `/index.html`.
- Route 53 A + AAAA aliases for IPv6 coverage.

### `landing` — marketing site
- Independent S3 bucket + CloudFront distribution at the apex `explainmyscanapp.com` + `www.<root>`. Separate cert; separate KMS-encrypted bucket; single static HTML file with the real logo at 88 px in the header.
- The frontend module's CSP gates work fine here too — apex CSP is much stricter (`default-src 'self'`) since the landing has no third-party scripts.

### `api` — API Gateway + WAF + custom domain
- REST API (not HTTP API — needed Cognito User Pool authorizer with WAF integration).
- WAF web ACL with the **AWS Managed Common Rule Set** + SQLi rule set. `SizeRestrictions_BODY` and `CrossSiteScripting_BODY` are explicitly overridden to count-only because they false-positive every multipart upload.
- Custom domain `api.dev.explainmyscanapp.com` with its own regional ACM cert and base-path mapping.
- Cognito JWT validation in the authorizer config — `identity_source = "method.request.header.Authorization"`.
- Per-stage CloudWatch access logging at JSON format (encrypted with the customer KMS key, 90-day retention).

### `processing` — the async pipeline + the read/export lambdas
- **Upload Lambda** (256 MB / 30 s): multipart parse, file-type whitelist, 10 MB cap, S3 put, DDB put with `status=processing`, SQS send with the doc tuple.
- **Process Lambda** (512 MB / 5 min): the heavy lifter. Branches on file type:
  - SVG → regex strip → Bedrock
  - PNG/JPG/GIF/WEBP → Textract `DetectDocumentText` (sync) → Bedrock
  - PDF → `pypdf` text extraction; if empty (scanned PDF), `StartDocumentTextDetection` → SNS callback → `GetDocumentTextDetection` → Bedrock
  - After Bedrock summary completes: `StartDocumentTextDetection` is not used here; instead the summary's diagram-description sentence is fed to Stability AI in us-west-2, then a second Claude pass calibrates label positions on the returned image.
- **Read Lambda** (256 MB / 15 s): handles GET /documents, GET /documents/{id}, GET /documents/{id}/download, DELETE /documents/{id}, POST /documents/{id}/retry, DELETE /account. Issues presigned URLs (SigV4 — required for KMS-encrypted objects).
- **Export Lambda** (1024 MB / 60 s, isolated from read so the hot path stays cheap): builds the per-doc PDF + zip in /tmp using `fpdf2`, uploads to `s3://.../exports/{userId}/{exportId}.zip` with `Content-Disposition: attachment` set on the object metadata, returns a 1-hour presigned URL.
- **DLQ Processor Lambda** (256 MB / 30 s): SQS-triggered from the processing DLQ (after `maxReceiveCount=3`). Conditional `UpdateItem` flips stuck rows to `failed`, with `ConditionExpression: #s <> :complete` to dodge the race where a slow-but-successful run finishes after the DLQ message lands.

### `storage` — KMS, S3, DynamoDB
- **Customer-managed KMS key** with policy granting CloudTrail, CloudWatch Logs, SQS, SNS, S3 server-side encryption, and CloudFront (via OAC source ARN) decrypt. KMS key policy lives at the root module — putting it inside `storage` or `frontend` would create a module cycle.
- **Documents bucket**: KMS-encrypted, versioned, with a lifecycle that transitions to Glacier at 365 days then expires at 7 years (HIPAA audit retention). A second lifecycle rule expires `exports/` objects after 1 day. Public access fully blocked; logging into a separate access-logs bucket.
- **DDB table** with `userId` hash + `docId` range, a GSI `userId-uploadedAt-index` for time-ordered listing, KMS server-side encryption, PITR enabled, Streams `NEW_AND_OLD_IMAGES` for future audit pipelines.

### `monitoring` — CloudWatch, RUM, alarms
- **CloudTrail** trail capturing every management + data event on the documents bucket, KMS-encrypted, with 7-year S3 lifecycle.
- **Per-Lambda CloudWatch log groups** (KMS-encrypted, 90-day retention) — explicit resources so the retention isn't AWS's default 30 days.
- **DLQ depth alarm**: `ApproximateNumberOfMessagesVisible > 0` over 5 min → SNS → optional email subscription. Operator gets notified before users have to report.
- **Ops dashboard**: 6 widgets in a 24-column grid — SQS visible-messages (main + DLQ overlaid in red), Lambda errors at 5-min sum, Lambda duration p95, API Gateway 4xx/5xx + latency, DDB throttled requests + RCU/WCU.
- **CloudWatch RUM**: app monitor + Cognito identity pool (unauthenticated) + IAM role tightly scoped to `rum:PutRumEvents` on the monitor ARN. Client telemetries: errors, performance, http (with `recordAllRequests=false` so request bodies never get captured). URL pageId coarsening keeps document UUIDs out of event payloads.

### `notifications` — SES with full deliverability
- **Domain identity** for the apex, with `aws_ses_domain_dkim` + three Route 53 CNAMEs.
- **Custom MAIL FROM** at `bounce.<root>` with its own SPF TXT and MX → `feedback-smtp.<region>.amazonses.com`. Aligns the envelope-from with the visible From: header so DMARC's SPF half passes.
- **DMARC** TXT at `_dmarc.<root>` in monitor mode (`p=none; aspf=r; adkim=r`) — Gmail weights the *presence* of a DMARC opinion even at p=none.
- **Sandbox recipient identity** for dev testing (gets a verification email from AWS).
- **Friendly From** uses display-name format (`"ExplainMyScan" <noreply@…>`) plus **Reply-To** at `support@<root>`.

## Failure modes we designed for

| Failure | Detection | Recovery |
|---|---|---|
| Lambda OOM or timeout mid-processing | SQS visibility timeout → message redrives | After 3 attempts → DLQ → `dlq_processor` flips row to `failed` → user sees retry button |
| Textract job fails on a corrupt PDF | SNS callback delivers `Status=FAILED` | Process Lambda marks row failed with the Textract status as the error string |
| Bedrock label-calibration call rate-limits | `ClientError` in the second vision call | Ship the doc with diagram + image_description + zero labels (still useful), log warning |
| Stability content filter rejects the prompt | `ClientError` from `bedrock_image.invoke_model` | Ship the doc with summary + questions, no diagram. Logged but non-fatal. |
| User uploads + immediately closes the tab | Nothing breaks — pipeline runs to completion regardless | Completion email sent via SES; user can sign back in later |
| User's session expires mid-flight | Frontend `getIdToken()` proactively refreshes; if refresh fails, redirect to landing | Re-sign-in, doc still listed |
| User typos a new email in Settings | Cognito stays on the OLD email until verify | Old email keeps working, verification code goes to typo'd address (never confirmed), no lockout |
| SES domain not yet DKIM-verified | SES rejects send with `MessageRejected` | Logged warning; doc still complete in DDB; user can sign in to view |
| User deletes account mid-processing | Frontend wipes data first, then deletes Cognito user | If Cognito DeleteUser fails after S3+DDB wipe, user is still signed in and can retry — better than orphaning PHI |

## Observability surfaces

- **CloudWatch ops dashboard** for at-a-glance health: `https://us-east-1.console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:name=explainmyscan-ops-dev`
- **CloudWatch RUM** for client-side errors + page-load timing: `https://us-east-1.console.aws.amazon.com/rum/home?region=us-east-1#/monitors/details/explainmyscan-dev`
- **DLQ depth alarm** with optional email subscription
- **Per-Lambda log groups**, KMS-encrypted with the customer key, 90-day retention
- **API Gateway access log** with structured JSON (requestId, sourceIp, requestTime, httpMethod, resourcePath, status, latency)

## What's deliberately *not* built yet

- **Multi-region**: us-east-1 only. DDB is single-region, S3 has no replication. v1 launch decision — add it before scaling beyond one region's blast radius.
- **Dead-letter persistence**: DLQ messages live 14 days max (SQS limit). Long-term audit trail would copy them to S3 before expiry. Noted in the SQS module's comment; not implemented yet.
- **A separate prod environment**: dev is the only deployed workspace today. The Terraform is parameterized for it — `environment = "prod"` would stand up a parallel stack — but actually doing the apply (+ DNS shuffle for the SPA vs landing at the apex) is its own task.
- **Sign-up email customization**: Cognito's default sign-up verification email is bare. Custom SES sender is configured but the templates aren't yet swapped.
