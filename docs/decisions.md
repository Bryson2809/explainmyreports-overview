# Engineering decisions

Short ADR-style notes on the non-obvious calls — what I picked, what I rejected, and why. Roughly chronological.

## ADR-001 — FastAPI prototype before AWS rewrite
**Context.** First version needed to validate the *product* (does an AI summary of a real medical report actually feel useful?), not the architecture.

**Decision.** Single-file FastAPI app calling OpenAI directly, served on a developer's laptop with a vanilla-JS frontend.

**Why.** OpenAI was fastest for prototyping; the AWS BAA + Bedrock setup would be wasted time if the prompt didn't land. Iterating in one Python file with no infra is the right shape for prompt design.

**Consequence.** Generated a "golden output" for what the AI summary should look like before any infra was built. The Bedrock prompt later was tuned against that reference, not designed from scratch.

---

## ADR-002 — Bedrock Claude for analysis, Bedrock Stability for diagram
**Context.** Need a HIPAA-eligible LLM + a HIPAA-eligible text-to-image generator. Customer data flows through both.

**Decision.** Bedrock for everything. Claude Sonnet 4.6 via cross-region inference profile (`us.anthropic.*` → fans out to us-east-1 / us-east-2 / us-west-2). Stability Stable Image Core for the diagram, pinned to us-west-2 (the only region with an active text-to-image model on Bedrock as of build time).

**Why.** AWS BAA covers Bedrock. Going to OpenAI for analysis would require either OpenAI Enterprise (paid) or a separate BAA that's not as well-traveled. Mixing AWS BAA for one service and non-AWS for another would multiply paperwork and audit risk.

**Rejected.** Amazon Nova Canvas — marked LEGACY by AWS in 2026. Default us-east-1 Bedrock for image gen — no active text-to-image model there.

**Consequence.** The process Lambda uses two different boto3 clients with different regions (`bedrock = boto3.client("bedrock-runtime")` for default region, `bedrock_image = boto3.client("bedrock-runtime", region_name="us-west-2")` for Stability). The IAM policy is explicit about the resource ARNs across both regions, plus a `aws-marketplace:Subscribe` grant because Stability is billed via Marketplace and Bedrock InvokeModel requires the calling role to hold those actions even after the subscription is active.

**Update.** The diagram model was upgraded from Stable Image Core to **Stable Image Ultra** (same BAA + us-west-2 region, materially better anatomical fidelity). Generation now retries up to 3 times and pins a fixed non-zero seed so the same document yields a reproducible diagram.

---

## ADR-003 — Cognito User Pool with MFA-required, hosted UI *(superseded — see Update)*
**Context.** Need a HIPAA-aligned auth surface that supports MFA and OAuth.

**Decision.** Cognito User Pool with `mfa_configuration = "ON"` + TOTP, hosted UI for the sign-in flow, custom domain (`auth.dev.explainmyreports.com`), expo-auth-session (PKCE) on the client.

**Why.** Cognito is the AWS BAA path. Hosted UI saves implementing sign-in / sign-up / forgot-password flows. PKCE is the right OAuth pattern for a public client (no client secret). The custom domain is a branding win without significant cost or complexity once the parent's A-record prereq is satisfied.

**Rejected.** Auth0 (separate BAA needed; vendor lock-in concerns), Firebase Auth (no AWS BAA equivalent), implementing sign-up flows in the SPA (huge surface area for not much value).

**Notes.** The `aws.cognito.signin.user.admin` scope is the non-obvious requirement for SPA-side ChangePassword / DeleteUser / UpdateUserAttributes. Default `openid email profile` doesn't authorize those calls — Cognito returns "Access Token does not have required scopes" without it.

**Update — superseded by an in-app auth surface.** The hosted UI + OAuth-code-grant flow was replaced by a custom in-app flow that calls the Cognito IDP directly with `USER_PASSWORD_AUTH` (and handles the `SOFTWARE_TOKEN_MFA` challenge in-app). Owning sign-up / confirm / sign-in / forgot-reset lets the SPA branch on Cognito error codes — most importantly auto-resending the code and routing to the confirmation screen on `UserNotConfirmedException` instead of dead-ending. MFA also moved from required (`mfa_configuration = "ON"`) to **optional, self-service** (`"OPTIONAL"`) — enabled or disabled from Settings. The hosted-UI Terraform still exists but is no longer in the request path. See ADR-004, which now covers the whole auth surface rather than just the Settings RPCs.

---

## ADR-004 — Don't use aws-amplify for Cognito
**Context.** SPA needs to call ChangePassword, GetUser, DeleteUser, UpdateUserAttributes, VerifyUserAttribute from Settings.

**Decision.** Hand-rolled fetch wrappers (40 lines in `app/lib/cognito.ts`).

**Why.** aws-amplify is ~400 KB minified, brings in a huge dependency tree, and adds zero functionality beyond what `fetch + X-Amz-Target` headers already give you. The Cognito IDP is just JSON-over-HTTPS with a header.

**Consequence.** The web bundle is ~400 KB smaller. Pattern is the same across all five RPCs — the wrapper is dead simple.

**Update.** When auth moved in-app (see ADR-003), this same hand-rolled approach scaled to the *entire* auth surface — `InitiateAuth` (USER_PASSWORD_AUTH), the `SOFTWARE_TOKEN_MFA` challenge, sign-up, confirm, resend, and forgot/reset — not just the Settings RPCs. Still no aws-amplify; `app/lib/cognito.ts` keys off the Cognito `__type` error codes so screens can branch (e.g. resend confirmation, redirect to MFA).

---

## ADR-005 — CloudWatch RUM, not Sentry
**Context.** Frontend needs real-user error + perf monitoring. Sentry is the default; it has the better UX.

**Decision.** CloudWatch RUM.

**Why.** Sentry's BAA tier requires an enterprise contract. RUM is BAA-covered under the existing AWS agreement. For a solo project not yet revenue-generating, "free under existing BAA" beats "$$$/month + new BAA paperwork."

**Cost of the trade.** RUM's UX is rougher (less polished error triage, no source-map upload at parity, weaker release tracking). Acceptable for v1; revisit if Sentry's BAA terms get more accessible.

**Notes.** PHI guardrails in the client config: `pageIdFormat: "PATH"` (coarsen the URL), `pagesToExclude: [/\/auth\/callback/]` (drop the OAuth callback's session-scoped params), `http` telemetry with `recordAllRequests: false` (no request bodies — they could carry document text).

---

## ADR-006 — Separate `read` and `export` Lambdas
**Context.** The CCPA/GDPR export endpoint needs more memory (1024 MB) and a longer timeout (60 s) than the rest of the read API (256 MB / 15 s).

**Decision.** A dedicated `export` Lambda, separate from `read`.

**Why.** Bumping the read Lambda's memory triples the per-invocation cost of *every* GET /documents and GET /documents/{id}. Those are the hot path — they should stay cheap. The export endpoint is rare and tolerates a slightly heavier config.

**Rejected.** One unified Lambda with shared config (cost penalty), two separate Lambdas with a shared library layer (more terraform, more zip-build coordination, marginal benefit).

---

## ADR-007 — DLQ message → DDB row state machine
**Context.** SQS retries the processing message 3× before redriving to a DLQ. Without explicit handling, the DDB row stays in `extracting` or `analyzing` forever and the user sees an infinite spinner.

**Decision.** Dedicated `dlq_processor` Lambda triggered by the DLQ. Marks the corresponding row `failed` with a descriptive error. Conditional on `status <> 'complete'` to avoid clobbering a successful-but-slow run.

**Why.** Closes the silent-stuck-doc gap. Lets the frontend's polling UI flip to the failed-state card (with a retry button) instead of spinning forever.

**Notes.** The race: a slow Bedrock call eventually returns and completes the doc *after* the SQS message has already been moved to the DLQ. Without the `ConditionExpression`, the DLQ processor would mark a successful doc as failed. With it, the update no-ops.

---

## ADR-008 — Data export as ZIP of unified PDFs, not raw files or JSON
**Context.** GDPR/CCPA-style data portability. Tried three shapes.

**Decisions, chronologically.**
1. **JSON dump** with presigned download URLs. Implemented, PR opened, never merged.
2. **ZIP with per-doc folders** (`summary.md` + `original.<ext>` + `diagram.png` + `metadata.json`). Better but still developer-shaped output.
3. **ZIP with one PDF per doc** (`summary.pdf` + `original.<ext>`). Summary PDF has the analysis text, the diagram inline, and the questions list. Final shipped form.

**Why the iteration mattered.** Each version surfaced a real concern. JSON works if the recipient is a developer or another system; useless for a patient. Folder-with-multiple-files is browsable but requires the user to know what a JSON file is. PDF is the universal "I can open this in any program" answer.

**Implementation notes.** `fpdf2` instead of WeasyPrint (no system deps), default Helvetica with latin-1 transliteration for the smart quotes and em-dashes that Sonnet emits, emojis stripped (they're already in section titles which get rendered as plain caps). The diagram is written to /tmp temporarily so `pdf.image()` can read it. ~13 MB zipped lambda bundle.

**Update — PDF generation moved to headless Chromium.** The `fpdf2` reconstruction never quite matched the website's detail page. It was replaced by a dedicated `pdf-render` Lambda that renders each PDF from the *same* HTML/CSS as the detail page using **headless Chromium via Playwright**, so the export is page-faithful. Rendering is idempotent on a canonical S3 key and triggered by SNS when a doc completes (or synchronously by the read Lambda to backfill older docs); the export Lambda just zips the resulting per-doc PDFs.

---

## ADR-009 — Custom MAIL FROM for SPF alignment
**Context.** Completion emails from `noreply@explainmyreports.com` were landing in Gmail spam even though SES delivered cleanly and DKIM passed.

**Decision.** Configure a custom MAIL FROM domain at `bounce.explainmyreports.com` with its own SPF TXT and MX → `feedback-smtp.<region>.amazonses.com`.

**Why.** Without a custom MAIL FROM, SES uses `*.amazonses.com` as the envelope-from. The visible From: domain (`explainmyreports.com`) and the envelope-from domain (`amazonses.com`) don't align, so DMARC's SPF half fails. DKIM still passes, the email still delivers — but Gmail downgrades the trust score and routes to spam.

With the custom MAIL FROM, both domains align. DMARC passes both halves.

**Side note.** Adding DMARC at all — even at `p=none` (monitor mode) with no `rua` URI — was a measurable deliverability improvement. Gmail and Yahoo weight the *presence* of a DMARC policy as a positive signal regardless of how lax the policy is.

---

## ADR-010 — Lambda packaging via build dir
**Context.** Terraform's `archive_file` zips a directory. The directory needs to be populated before each apply, or the lambda ships stale code with no warning.

**Decision.** `lambdas/build.sh` script that wipes `lambdas/build/<name>/` and re-stages from source on every run.

**Why.** A dedicated step makes the staleness explicit. Running it is a memorable habit; not running it surfaces immediately in a hash diff.

**The bug that motivated this.** Shipped a release where the build dir was a week stale. The lambda hash changed in the terraform plan (because the zip was rebuilt by terraform), but the *content* was a week behind. Symptoms: a recently-shipped feature stopped working with no errors in CloudWatch — the deployed code didn't have the feature at all. Took a CloudWatch log dive to figure out.

**Notes.** The pip flags pin to Linux + Python 3.12 regardless of the local interpreter:
```
--platform manylinux2014_x86_64 --only-binary=:all: --python-version 3.12 --implementation cp
```
This means a developer on macOS Python 3.14 can still produce a lambda-runnable zip without installing Python 3.12 locally.

---

## ADR-011 — Mark `lambdas/build/` and `terraform/modules/processing/build/` ignored
**Decision.** Both are in `.gitignore`. Source-of-truth is the `.py` files under `lambdas/<name>/` + the build script.

**Why.** Generated artifacts shouldn't be in source control. They're large, they churn on every dependency change, and they'd show up in every diff as noise.

**Consequence.** First-time setup on a new worktree needs `bash lambdas/build.sh` before `terraform plan`. Documented in CLAUDE.md so future-me doesn't trip on it.

---

## ADR-012 — Per-doc Cognito user-pool email verification before update
**Context.** Settings has a "Change email" flow. Without explicit configuration, Cognito updates the email attribute immediately, marks it unverified, and the user is locked out if they typo the new address (token refresh + recovery both go to the typo'd mailbox).

**Decision.** Set `user_attribute_update_settings.attributes_require_verification_before_update = ["email"]` on the user pool. Cognito sends a verification code to the new address; the OLD email keeps working as the sign-in identity until the code is confirmed.

**Why.** Defensive default. The user's password recovery flow is tied to their verified email — letting an unverified email become the active one is a footgun.

---

## What I'd do differently in v2

- **Build the prod environment alongside dev from the start.** Single-env Terraform feels simpler but doesn't reveal the prod-specific work (apex SPA vs landing collision, separate KMS, separate Cognito) until you go to stand up prod. Better to scaffold both early. *(Done — a prod stack now runs at `app.explainmyreports.com`, with shared-singleton resources gated to dev's state and referenced by prod.)*
- **Pick Sentry once it's affordable.** RUM works but its triage UX really is rough.
- **EAS Build hooked up for native builds early.** The Expo Router native paths exist but I've only ever tested them via `expo start`. EAS Build for iOS/Android is a few config knobs but worth doing before the codebase grows.
- **Real CI.** *(Done — GitHub Actions now runs pytest + `tsc --noEmit` + `terraform validate` as the merge gate, with deploys triggered via `workflow_dispatch` and OIDC federation. The early PRs shipped without it; CI landed as the project grew past the first dozen.)*
