# Dev to Prod Fast

## Prerequisites

- The GitHub App in your current setup is providing authorization, not cryptographic identity. What The GitHub App Gives You is Fine-grained installation token: Scopes restricted to exactly what the App was granted (e.g. contents: read/write for a repo). Short‑lived (usually 1 hour) → automatic rotation each workflow run. Separation of privileges:
  You don’t leak a long‑lived PAT tied to a user.
  Revoking the App or uninstalling instantly kills future automation without touching user credentials.
  Auditability:
  Pushes made with the installation token show in GitHub’s internal audit trail as originating via an App installation.
  Principle of least privilege:
  An App can be limited to specific repositories; a PAT usually spans all repos the user can access unless replaced by fine‑grained PATs.
  Operational safety:
  If compromised, blast radius is smaller (scoped + ephemeral).
  CI convenience:
  No need to manage manual rotation or expiration like with a PAT or SSH deploy key.
  What It Does NOT Provide
  Verified commit signature (no GPG/SSH identity; Apps can’t upload keys).
  Semantic meaning of “who” authored the commit for cryptographic purposes.
  Automatic conventional commit enforcement (that’s separate tooling).
  Release creation logic (unless you implement or use other actions).
  Commit signing—still needs a user or bot account key if you want the “Verified” badge.
  Why Still Use It Even If Not Signing
  You want automation that can write files (like the date file) and push without granting a broadly powerful token.
  You may later expand the App to do more restricted operations (open issues, label PRs, etc.) while keeping risk low.
  You avoid checking in a PAT that might accidentally gain more scope over time.
- GitHub user is required as GitHub App cannot produce Verified GPG signatures. Verified commit signatures are tied to a GitHub user (or org bot account) that has uploaded the public key. Apps don’t have a profile to host a GPG or SSH signing key.
  A GitHub user (human account or dedicated bot account) supplies the long‑term identity layer that the platform ties trust, authorship, and ownership to. Here’s what that user identity is used for and why it’s distinct from a GitHub App:

Core Roles of a GitHub User Identity
Authorship and Attribution

Commits, issues, PRs, comments are attributed to a user.
The profile acts as the canonical source of “who did what” across repositories.
Cryptographic Trust Anchors

Public GPG and SSH signing keys are uploaded to the user profile.
GitHub verifies commit signatures by matching the signature’s key fingerprint to keys stored on the user’s account and the commit’s author/committer email to one of the user’s verified or noreply emails.
Only user (or org bot) accounts can host these keys; Apps cannot.
Email / Identity Binding

Verified emails (or automatically generated noreply aliases) connect commit metadata to the user account.
Changing (or revoking) an email affects future commit verification logic.
Token Generation

Personal Access Tokens (PATs) or fine‑grained PATs originate from a user account.
These represent delegated capabilities (scoped or broad) and carry the user’s authority.
Ownership and Membership

Users belong to organizations, teams, and can be granted repo roles (admin, maintainer, triage, etc.).
Access control and audit reasoning pivot around users (and org structures).
Reputation and Social Signals

Contribution graphs, followers, stars, sponsorships, and activity feeds center on user accounts, not Apps.
Security Controls

2FA / passkey enforcement, account recovery, session management, SSH key rotation, GPG key rotation—these are user account level functions.
Security advisories, notifications, and watch settings are all per-user.
Why a Bot User (Machine Account) Sometimes Exists
Separation of automated activity from human commits (clear audit trail).
Stable identity for signing keys and automation.
Revokable independently; rotate keys or credentials without affecting human developer tokens.
Avoids mixing CI commit authorship with personal developer graphs.

A single org-wide GitHub App plus one shared bot can work, but at your scale (1600 repos, 500 devs, many independent teams) it’s rarely optimal without segmentation. It becomes a central dependency and potential bottleneck. A hybrid model (core shared assets + scoped variants) usually wins on resilience, autonomy, and least privilege.

Decision Factors
Autonomy vs Central Control

Single App/Bot: Faster initial rollout, consistent behavior, easier policy enforcement.
Multiple Apps/Bots: Teams tailor permissions, versioning rules, release cadence, and failure isolation.
Blast Radius / Risk

One App private key compromise impacts every repo it’s installed on.
Segmented apps limit exposure (e.g., per domain, per criticality tier).
Permissions & Scope

Different repos may need different scopes (packages, deployments, issues, etc.). One “superset” app violates least privilege.
Release Semantics

Libraries, services, CLI tools, frontends may require different semantic-release setups (prereleases, channels, tag formats). Centralized config can become cluttered.
Team Velocity

Independent teams resent slow centralized change control. Decentralization empowers experimentation.
Key & Identity Management

One GPG key for a shared bot signing all release commits/tags makes attribution opaque and raises trust questions.
Multiple bot identities allow domain-level provenance and simpler key rotation.
Operational Load

Single bot: simpler monitoring but larger queue of feature requests.
Multiple: more assets to maintain, but changes are localized.
Recommended Architecture Pattern
Layered model:

Layer 1: Shared “Golden” Reusable Workflows / Composite Actions

A repo (e.g. org/automation-foundation) hosting:
Reusable workflow for semantic versioning (inputs: release type strategy, tag prefix, monorepo path).
Composite action for generating GitHub App token.
Shared commit signing helper (optional).
Standard conventional commit check.
Layer 2: GitHub Apps Segmentation

App A: Core release automation for service repos (contents + metadata + packages).
App B: Library releases (needs packages: write).
App C: Frontend deployment tasks (pages, environments).
App D: Experimental/sandbox (broad but isolated).
Install only where needed; do NOT give one App broad org-wide scope unless strictly required.
Layer 3: Bot Accounts (Machine Users)

One per domain cluster (e.g., infra-bot, services-release-bot, libs-release-bot).
Each has:
Its own GPG key (Ed25519, no expiry, rotated annually or semi-annually).
Noreply email tied to the account for Verified commits/tags.
Keys stored as secrets per repo or distributed via org-level secret inheritance (if using GitHub Enterprise).
Layer 4: Policy Enforcement

Org-level branch protection: require signed commits/tags for release branches.
Conventional commit validation via status check (composite action).
Optionally enforce verified signatures only on tags (lighter friction).
Semantic Versioning Strategy
Provide a base semantic-release config with extensible plugin slots:
Core: commit-analyzer, release-notes-generator, changelog, GitHub.
Optional: npm publish / container image tagging / artifact upload.
Allow teams to override:
tagFormat, preRelease, branches.
Keep config in a shared module @org/semrel-config (npm private package or raw file fetched via actions/checkout + path reference).
Use reusable workflow:
workflow_call inputs: publish: boolean, sign_tags: boolean, tag_prefix, package_manager.
Commit & Tag Signing
Option choices:

A. Sign only releases (tags) with bot keys (recommended minimal friction).
B. Sign all automated commits (can be noisy).
C. Sign changelog + version bump commits only.
D. Use Sigstore (experimental future) for artifact provenance; still keep GPG for GitHub “Verified”.

Operational best balance: A + optional C.

Key management playbook:

Each bot key: Ed25519, comment includes purpose + creation date.
Store private key + (optional) passphrase secrets: LIBS_BOT_GPG_PRIVATE_KEY, LIBS_BOT_GPG_PASSPHRASE.
Rotation: generate new key, upload public, switch secret, revoke old, document fingerprint in audit log.
Single vs Multiple Bot Comparison
Dimension Single Bot Multiple Bots
Simplicity High Medium
Attribution granularity Low High
Rotation overhead Low Medium
Risk concentration High Lower
Policy flexibility Low High
Cultural fit (independent teams) Often poor Better
When a Single App/Bot Is Acceptable
Early phase / pilot rollout.
High internal alignment; uniform release process.
Low variation in repo types (e.g., many similar microservices).
Strong internal SRE team willing to be central gatekeeper.
You can start centralized then progressively split:

Phase 1: Single App + single bot for pilot set (say 50 repos).
Phase 2: Gather variation requirements → domain split.
Phase 3: Migrate high-risk or specialized repos to domain-specific App and bot.
Phase 4: Standardize metrics & dashboards (release success rate, time-to-tag).

Governance & Guardrails
Maintain a registry file (YAML) listing:
Repo name
Assigned App
Assigned bot
Release workflow version
Signing mode (tags | commits | none)
Periodic audit script: ensure each repo uses latest composite action version; report drift.
Add an incident playbook: if App key leaked → revoke + rotate; if bot key leaked → revoke + rotate + invalidate recent tags (re-tag if needed).
Practical Implementation Steps
Create foundational reusable workflow:
release.yml (workflow_call) with inputs for sign mode.
Write composite action sign-tag:
Imports GPG key, sets user.name, user.email, signs tag.
Provision first bot identity + key; test on 3–5 repos.
Add dashboards (GitHub API + internal tool) tracking release frequency and failures.
Define segmentation rules (permission needs, domain boundary).
Clone App with tailored permissions sets when divergence appears.
Recommendation Summary
Don’t lock into a single App/bot for all 1600 repos long-term. Start centralized for speed, but plan for segmentation by domain, permission scope, and trust boundaries. Sign tags (not every commit) with per-domain bot keys. Use reusable workflows to keep consistency without over-centralizing logic.

If You Want Next Actions
I can:

Draft a reusable workflow skeleton.
Provide a sample registry YAML.
Generate a composite action for tag signing.
Show rotation script outline.
Just tell me which artifact you want first (workflow, composite action, registry) and I’ll build it. Let me know your preference.

- Variables
  - GH_APP_ID
  - GIT_SIGN_BOT_NAME
  - GIT_SIGN_BOT_EMAIL
- Secrets
  - GH_APP_PRIVATE_KEY
  - GIT_SIGN_BOT_GPG_PRIVATE_KEY
  - GIT_SIGN_BOT_GPG_PASSPHRASE
