<div align="center">
  <img src="images/logo.png" alt="ExplainMyScan" width="180" />

  # ExplainMyScan

  **Turn medical documents into plain-English summaries — solo-built, HIPAA-aligned, AWS serverless.**

  [Live preview](https://explainmyscanapp.com) · [App (dev)](https://dev.explainmyscanapp.com)

</div>

> **About this repo.** The application source is private. This page is a curated portfolio overview — the architecture, the engineering decisions, and a tour of the UI. If you're hiring and want to see code, reach out at `bryson2809@gmail.com` and we can talk.

---

## What it is

Patients get medical reports they can't read. Insurance summaries. MRI findings. Lab panels. The text is technical, the layout is hostile, and the few sentences that matter are buried.

ExplainMyScan takes any PDF, scan, or image of a medical document and returns:
- A **plain-English summary** with severity rating
- An **AI-generated anatomical diagram** illustrating the affected area
- **Questions to bring to your doctor** at the next visit

It's not medical advice. It's a translation layer so patients walk into appointments better prepared.

<p align="center">
  <img src="images/01-landing.png" alt="ExplainMyScan landing page" width="100%" />
</p>

---

## Architecture

End-to-end serverless on AWS. One developer, one production-ready stack.

```mermaid
flowchart LR
    subgraph Client["Browser / Native (Expo)"]
        UI[React Native + Web SPA]
    end

    subgraph Edge["CloudFront + Route 53"]
        CF[CloudFront]
        DNS[Route 53]
    end

    subgraph Auth["Cognito"]
        Pool[User Pool<br/>MFA required]
        HUI[Hosted UI<br/>auth.dev.explainmyscanapp.com]
        IDP[Identity Pool<br/>unauth → RUM]
    end

    subgraph API["API Gateway + WAF"]
        GW[REST API<br/>api.dev.explainmyscanapp.com]
    end

    subgraph Lambda["Lambda Functions"]
        Upload[upload]
        Process[process]
        Read[read]
        Export[export]
        DLQ[dlq_processor]
    end

    subgraph AI["Bedrock"]
        Claude[Claude Sonnet<br/>analysis + vision]
        Stability[Stability AI<br/>diagram gen]
    end

    subgraph Storage["S3 + DynamoDB"]
        Docs[(S3: documents<br/>KMS encrypted)]
        DDB[(DynamoDB: index<br/>PITR enabled)]
    end

    subgraph Queue["SQS + SNS"]
        Main[Processing Queue]
        Dead[DLQ]
        Tex[Textract Topic]
    end

    subgraph Obs["Observability"]
        RUM[CloudWatch RUM]
        Logs[CloudWatch Logs]
        Alarms[Alarms → SNS → SES]
    end

    subgraph Mail["SES"]
        SES[SES<br/>+ DKIM + SPF + DMARC<br/>+ custom MAIL FROM]
    end

    UI --> CF
    UI --> Pool
    UI --> HUI
    UI --> IDP
    IDP --> RUM
    CF --> GW
    GW --> Upload
    GW --> Read
    GW --> Export
    Upload --> Docs
    Upload --> DDB
    Upload --> Main
    Main --> Process
    Process --> Tex
    Tex --> Process
    Process --> Claude
    Process --> Stability
    Process --> Docs
    Process --> DDB
    Process --> SES
    Main -.failed 3x.-> Dead
    Dead --> DLQ
    DLQ --> DDB
    Dead --> Alarms
    Alarms --> SES
    Export --> Docs
    Export --> DDB
    Read --> Docs
    Read --> DDB
```

### Document processing pipeline

```mermaid
sequenceDiagram
    participant U as User
    participant API as API Gateway
    participant Up as Upload λ
    participant S3 as S3
    participant DDB as DynamoDB
    participant SQS as SQS
    participant Pr as Process λ
    participant Tx as Textract
    participant Br as Bedrock
    participant SES as SES

    U->>API: POST /documents (multipart)
    API->>Up: invoke
    Up->>S3: PutObject (original)
    Up->>DDB: PutItem (status=processing)
    Up->>SQS: SendMessage
    Up-->>U: 201 {docId}

    SQS->>Pr: invoke (async)
    Pr->>DDB: status=extracting
    Pr->>Tx: StartDocumentTextDetection
    Tx-->>Pr: SNS callback (async)
    Pr->>DDB: status=analyzing
    Pr->>Br: Claude — analysis
    Pr->>Br: Stability — diagram
    Pr->>Br: Claude — label calibration
    Pr->>S3: PutObject (diagram)
    Pr->>DDB: status=complete + summary
    Pr->>SES: SendEmail (notify user)

    U->>API: GET /documents/{id} (poll)
    API->>DDB: GetItem
    DDB-->>U: status + summary + presigned URLs
```

### Auth flow

```mermaid
sequenceDiagram
    participant U as User
    participant SPA as SPA (expo-auth-session)
    participant HUI as Hosted UI<br/>auth.dev.explainmyscanapp.com
    participant Pool as Cognito User Pool
    participant API as API Gateway

    U->>SPA: click "Sign in"
    SPA->>HUI: OAuth Authorize (Code + PKCE)
    HUI->>U: branded sign-in form
    U->>HUI: email + password + TOTP
    HUI-->>SPA: redirect with code
    SPA->>HUI: POST /oauth2/token (code + verifier)
    HUI-->>SPA: id_token + access_token + refresh_token
    SPA->>API: GET /documents (Authorization: id_token)
    API->>Pool: validate JWT (Cognito authorizer)
    API-->>SPA: 200
```

---

## Engineering decisions worth talking about

### 1. HIPAA without the HIPAA tax
Everything that touches PHI runs under AWS's BAA: Cognito, Lambda, S3, DynamoDB, SQS, SNS, Bedrock (Anthropic + Stability), Textract, SES, CloudWatch (including RUM). Customer-managed KMS key encrypts everything at rest — bucket, table, queue, log group. MFA-required Cognito with TOTP, CloudTrail with 90-day retention, WAF in front of API Gateway.

The non-obvious move: **CloudWatch RUM instead of Sentry**. Most real-user-monitoring vendors require an enterprise contract for HIPAA-eligible BAA coverage; RUM is BAA-covered by default and the client SDK supports URL coarsening to keep PHI proxies (document UUIDs in path params) out of the event stream. Trade smaller feature set for zero additional BAA paperwork.

### 2. The two-call vision pipeline
The diagram needs labels (numbered callouts pointing at anatomical features) that match the user's specific findings. Naive approach: have one model generate both image + labels. That fails — the image generator doesn't know what it drew at pixel-level precision.

Actual approach:
1. **Claude generates** a structured analysis with a `🧠 VISUAL DIAGRAM DESCRIPTION` line (one prose sentence for the image model) and a `📍 LABELS` section (short labels paired with anatomical references).
2. **Stability generates** the image from the description.
3. **Claude (vision)** receives the generated image and the labels, and returns `{x, y}` percentages for each label.

The third call only runs after the image exists. If it fails (rate limit, transient error), the document still ships with diagram + image_description + zero labels rather than blocking on the visual pass.

### 3. DLQ race condition
SQS retries the processing message 3× before sending it to the DLQ. A dedicated `dlq_processor` Lambda catches DLQ messages and marks the corresponding DDB row `status=failed` so the user's polling UI flips to the retry button instead of spinning forever.

**The race**: a slow-but-eventually-successful processing run can complete *after* the DLQ message lands. Without a guard, the DLQ processor would clobber a successful `status=complete` back to `failed`.

**The guard**: every DDB update from the DLQ processor uses a `ConditionExpression: #s <> :complete`. If the doc finished first, the update no-ops and the user sees their successful summary.

### 4. Email deliverability is half DNS, half headers
Cognito + AWS SES + custom domain is the easy part. Getting to inbox instead of spam took five separate things:
- **DKIM** CNAMEs from SES auto-set up
- **SPF** TXT at apex (`v=spf1 include:amazonses.com -all`)
- **DMARC** at `_dmarc` (monitor mode, `p=none; aspf=r; adkim=r`) — Gmail weights *presence* of DMARC even at p=none
- **Custom MAIL FROM domain** at `bounce.explainmyscanapp.com` with its own SPF + MX so the envelope-from and From: header domains align (without this, SPF passes but DMARC's SPF half fails because of misalignment)
- **Display-name From header** (`"ExplainMyScan" <noreply@…>`) and **Reply-To** (`support@…`)

First-email-after-DKIM landed in spam. Second one, after these landed, hit inbox.

### 5. Lambda packaging: build dir vs source
Terraform's `archive_file` zips a directory. The directory has to be populated before `terraform apply` or it ships stale code. There's no warning — the plan shows a hash change, but doesn't tell you the *content* is six versions behind.

The fix is a `lambdas/build.sh` script that wipes and repopulates `lambdas/build/<name>/` with the current source + pinned wheels (`--platform manylinux2014_x86_64 --python-version 3.12 --only-binary=:all:`). The hash drifts only when source or deps actually change. Bit me once before I figured it out — wrote it up so future-me wouldn't repeat the mistake.

### 6. Account export: JSON → ZIP → PDF
v1: JSON dump of metadata + presigned URLs. Technically correct, useless to a non-developer.

v2: zip with per-doc folders, but the summary was a markdown file and the diagram was a separate PNG.

v3 (shipped): zip with one **unified PDF per doc**. Title, formatted summary, diagram embedded inline, illustrative-only disclaimer, numbered questions list, final disclaimer footer. Built with `fpdf2` (no Pillow systems deps), default Helvetica, latin-1 transliteration for the smart quotes and em-dashes Sonnet emits. ~13 MB zipped lambda, well under the 50 MB direct-upload cap.

Cost works out to ~$0.005 per typical 50 MB export (dominated by S3 → internet egress at $0.09/GB). Storage is bounded by a 1-day lifecycle rule on the `exports/` S3 prefix.

---

## What's shipped (so far)

| Surface | Highlights |
|---|---|
| **Marketing site** | Static HTML at the apex; brand-matched, 88 px logo header, Terms + Privacy footer |
| **Auth** | Cognito User Pool, MFA-required TOTP, branded hosted UI at a custom subdomain, verify-before-update on email |
| **Upload** | PDF / image / SVG, XHR-with-onprogress (fetch can't surface upload progress), 10 MB cap, WAF in front |
| **Processing** | Async via SQS, Textract OCR for scans, Bedrock Claude for analysis, Bedrock Stability for diagram, two-call vision for label calibration |
| **Document UX** | Status polling, severity banner, AI-section markdown rendering, label callouts on the diagram, "longer than usual" hint after 2 minutes |
| **Failure recovery** | DLQ processor flips stuck rows to `failed`, in-app retry button re-enqueues the SQS message, SNS-driven email alert to the operator |
| **Settings** | Change password, change email (2-step verification), MFA status, delete account (wipes S3 + DDB + Cognito), download all data (ZIP of unified PDFs) |
| **Onboarding** | 3-step consent gate (welcome → privacy promise → terms acceptance), versioned acceptance so material terms changes re-prompt |
| **Legal** | /terms and /privacy public routes with DRAFT banner; placeholder copy intended for counsel review |
| **Observability** | CloudWatch RUM (client errors + page-load + http), CloudWatch dashboard (queue depth, lambda errors, API 4xx/5xx, DDB throttles), SNS-fanned DLQ alarm |
| **Email** | SES with DKIM + SPF + DMARC + custom MAIL FROM; completion notifications, alert subscriptions |
| **Tests** | 81 pytest cases with moto for AWS service mocking and unittest.mock for Textract + Bedrock |
| **Infra** | Terraform with remote S3 state + DynamoDB lock table; 8 modules (auth, api, frontend, landing, processing, storage, monitoring, notifications) |

---

## UI tour

### Marketing landing
<p align="center"><img src="images/01-landing.png" alt="Marketing landing page" width="100%" /></p>

Branded apex domain. The logo image was extracted from the user's design, alpha-threshold-trimmed to remove anti-aliased edge pixels that were rendering as awkward whitespace, then squared with minimal padding.

### Branded Cognito sign-in
<p align="center"><img src="images/02-cognito-signin.png" alt="Cognito hosted UI" width="60%" /></p>

Hosted at `auth.dev.explainmyscanapp.com` (custom domain with ACM cert + Route 53 alias to Cognito's CloudFront). The Cognito API caps `imageFile` at ~96 KB raw / 128 KB base64; the 1024×1024 master logo gets downscaled to 224 px for this surface.

### App landing (signed-out)
<p align="center"><img src="images/03-app-landing.png" alt="In-app landing" width="100%" /></p>

The React Native + Web version of the landing, rendered from the same codebase that powers the native app target.

### Legal pages
<p align="center"><img src="images/04-legal-page.png" alt="Terms of Service page" width="100%" /></p>

Public `/terms` and `/privacy` with prominent DRAFT banner — copy is a placeholder pending counsel review, but the routing, content, and consent-gate plumbing are real.

### Onboarding consent gate
<p align="center"><img src="images/08-onboarding.png" alt="Onboarding flow" width="100%" /></p>

Three steps on first sign-in: welcome → privacy promise → terms acceptance. Acceptance is versioned (`TERMS_VERSION = "v1-2026-05-18"`); bumping it re-prompts every user. Signing out clears the local record so a different user on the same device gets prompted.

### Dashboard
<p align="center"><img src="images/05-dashboard.png" alt="Documents dashboard" width="100%" /></p>

Document list with status badges. Polling refreshes every 3 s for any doc that isn't in a terminal state. Pull-to-refresh on touch devices. Optimistic delete with rollback on error.

### Document detail
<p align="center"><img src="images/06-detail.png" alt="Document detail page with AI summary, diagram, and questions" width="100%" /></p>

*Screenshot uses a synthetic medical report — no real patient, provider, or examination.*

Severity banner at the top with a 1–4 visual scale and a generated explanation of why the level was chosen. The diagram embeds numbered callouts that position over the image at percentage-coordinates calibrated by a separate Claude vision pass. Each section header is an emoji + ALL-CAPS title pulled directly from Sonnet's structured output. Questions render as a numbered list at the bottom.

### Settings
<p align="center"><img src="images/07-settings.png" alt="Settings page with account, MFA, change email, change password, legal, data export, danger zone" width="100%" /></p>

Account email (read-only), MFA status (read-only — TOTP enrolled at sign-up), 2-step email change, password change, links to legal pages, data export (downloads a ZIP of per-doc PDFs), delete account (wipes S3 + DDB + Cognito).

---

## Tech stack at a glance

**Languages** — Python 3.12 (Lambda), TypeScript (frontend), Bash (deploy scripts), HCL (Terraform)

**Frontend** — Expo (React Native + Web), React 19, Expo Router, NativeWind v4 (Tailwind on RN), TanStack Query, expo-auth-session (PKCE OAuth), expo-secure-store, react-native-svg, aws-rum-web

**Backend** — AWS Lambda (Python), boto3, pypdf, fpdf2

**AI** — AWS Bedrock: Claude Sonnet 4.6 via cross-region inference profile (analysis + vision), Stability Stable Image Core (text-to-image diagram)

**Infrastructure** — Terraform with remote state, AWS Cognito, API Gateway (REST + Cognito authorizer + WAF), CloudFront, Route 53, ACM, KMS, S3 (KMS, versioning, lifecycle, OAC), DynamoDB (KMS, PITR, GSI, Streams), SQS (with DLQ), SNS, SES (DKIM + SPF + DMARC + custom MAIL FROM), Textract, Bedrock, CloudWatch (Logs, Dashboard, Alarms, RUM), CloudTrail

**Testing** — pytest, moto, unittest.mock — 81 unit tests covering Lambdas including cross-user isolation, race conditions, malformed input handling

**Tooling** — gh CLI for PR workflow, git worktrees (main checked out separately for clean reference), Chrome DevTools MCP for in-loop UI verification

---

## A note on shipping pace

This was built solo, with frequent merges to keep the dev environment representative. From the first AWS commit to the current feature-complete state, **22 PRs** landed — most under an hour from branch to merge, each with tests and a clean test plan. Examples worth reading:

- `feat(notifications): SES completion email on processing complete` — the one with the SES sandbox + IAM scoping that taught me about envelope-from alignment
- `feat(reliability): DLQ processor, retry endpoint, frontend retry button` — closes the silent-stuck-doc gap end-to-end
- `feat(monitoring): CloudWatch RUM for the web app` — picked over Sentry to stay inside the BAA, with PHI-safe URL coarsening
- `feat(account): self-service data export delivered as a ZIP` — the v1 was JSON, v2 was a folder zip, v3 (shipped) is a unified PDF per doc

---

## Contact

If you've made it this far and want to talk — **bryson2809@gmail.com**.
