# 🚀 Versioning (Release) Reference Template

This repository demonstrates how to make modern, safe, and automated versioning easy to understand and adopt. It shows how any team can go from a manual, error-prone "version bump and tag" workflow to a fully automated release pipeline that handles versioning, tagging, changelogs, container publishing, and signing - all in a transparent and auditable way.

The aim is to help engineering teams:

- Build trust in automation
- Deliver software faster and more safely
- Follow consistent engineering practices that scale across NHS products

Whether you're a developer, tester, or tech lead, this repository shows how you can go from _commit → version → deploy → release_ without ever losing confidence in what's running in production.

This approach also lays the foundation for [feature toggling](https://github.com/NHSDigital/software-engineering-quality-framework/blob/main/practices/feature-toggling.md) where new functionality is deployed but not immediately exposed to users. Versioning provides the traceability and control needed to manage these toggled changes safely, allowing teams to ship small, reversible updates, test them in production, and enable them gradually when ready.

Beyond automation and feature management, this template also provides base for secure software supply chain practices, including artefact hardening and provenance. By producing signed, tamper-evident build outputs (such as container images and packages) with a clear chain of custody, teams can:

- Strengthen cyber resilience by ensuring only verified and trusted artefacts reach production
- Improve incident response through auditable traceability from code to deployed artefact
- Reduce the risk of supply-chain compromise by verifying every component's source, build process, and integrity

This aligns directly with NHS ongoing work to strengthen the security posture of workloads and artefact provenance, ensuring that every deployment is both secure by design and provable by evidence.

> [!IMPORTANT]
> This repository is not intended to be used standalone. It should be built on top of and complement the [NHS Repository Template](https://github.com/nhs-england-tools/repository-template), which defines the required baseline structure and configuration for all new repositories within the organisation.

- [🚀 Versioning (Release) Reference Template](#-versioning-release-reference-template)
  - [Overview](#overview)
  - [Structure](#structure)
    - [Repository files](#repository-files)
    - [How it works (plugin flow)](#how-it-works-plugin-flow)
    - [Configuration](#configuration)
      - [Variables](#variables)
      - [Secrets](#secrets)
  - [Prerequisites](#prerequisites)
    - [GitHub App setup](#github-app-setup)
    - [Bot setup for commit signing](#bot-setup-for-commit-signing)
    - [Container image signing with Cosign](#container-image-signing-with-cosign)
    - [Build provenance attestation](#build-provenance-attestation)
  - [Design decisions and rationale](#design-decisions-and-rationale)
    - [🧩 Why use a GitHub App Token instead of a Personal Access Token (PAT)](#-why-use-a-github-app-token-instead-of-a-personal-access-token-pat)
    - [🔐 Why the signing key belongs to a bot, not the App](#-why-the-signing-key-belongs-to-a-bot-not-the-app)
    - [🔏 Why we don't force tag signing by default](#-why-we-dont-force-tag-signing-by-default)
    - [📘 Why we don't commit a `CHANGELOG.md`](#-why-we-dont-commit-a-changelogmd)
    - [🐳 Use a flat registry for multiple images](#-use-a-flat-registry-for-multiple-images)
      - [Why this pattern works](#why-this-pattern-works)
      - [Alternative patterns considered](#alternative-patterns-considered)
      - [Detailed configuration](#detailed-configuration)
  - [How to use this repository](#how-to-use-this-repository)
    - [Adding a new feature](#adding-a-new-feature)
    - [How Conventional Commits affect versioning](#how-conventional-commits-affect-versioning)
    - [Final thought](#final-thought)
  - [Outstanding](#outstanding)

## Overview

On every push or merged pull request to the `main` branch, this workflow:

1. Authenticates securely using a short-lived GitHub App token
2. Reads commit messages that follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
3. Calculates the next semantic version
4. Updates a simple `VERSION` file with that number and signs the commit
5. Creates a Git tag such as `v1.2.3`
6. Publishes a GitHub Release entry with automatically generated release notes
7. Logs in to GitHub Container Registry (GHCR)
8. Builds and pushes a versioned container image

👉 **Result**: every version is predictable, traceable, and fully automated, with no need to manually tag, bump, or write changelogs.

## Structure

### Repository files

| File                                    | Purpose                                                                                                                 |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `.github/workflows/cicd-2-publish.yaml` | The core CI/CD workflow that performs the authenticated release, signing, and container publishing                      |
| `.releaserc`                            | Defines how `semantic-release` behaves, which plugins to use, what rules decide version bumps, and how commits are made |
| `VERSION`                               | A plain-text file containing the current version number (automatically maintained by the workflow)                      |

### How it works (plugin flow)

`semantic-release` runs as a series of stages, each handled by a plugin:

1. `@semantic-release/commit-analyzer` looks at your commit messages and decides whether this is a patch, minor, or major bump
2. `@semantic-release/release-notes-generator` writes clear, human-readable notes based on your commits
3. `@semantic-release/exec` updates the `VERSION` file in your repo with the new version number
4. `@semantic-release/git` commits that file change, signs it with GPG, and creates a tag
5. `@semantic-release/github` publishes the new version as a GitHub Release with those notes

That's it - a clean, logical pipeline that turns commit messages into traceable software releases.

### Configuration

#### Variables

Repository variables define the static configuration needed by the workflow:

- `GH_VERSIONING_APP_ID` - the numeric ID of the GitHub App
- `GIT_SIGNING_BOT_NAME` - display name used for the signed commits
- `GIT_SIGNING_BOT_EMAIL` - email address linked to the uploaded GPG key

#### Secrets

Repository secrets provide the credentials and cryptographic materials required to sign releases:

- `GH_VERSIONING_APP_PRIVATE_KEY` - the GitHub App's private key used to create short-lived auth tokens
- `GIT_SIGNING_BOT_GPG_PRIVATE_KEY` - private signing key of your release bot
- `GIT_SIGNING_BOT_GPG_PASSPHRASE` - the key passphrase
- `COSIGN_PUBLIC_KEY`
- `COSIGN_PRIVATE_KEY`
- `COSIGN_PASSWORD`

All of the above variables and secrets have an organisation-wide equivalent managed centrally by the NHS GitHub Admins. These defaults are automatically available to all repositories, ensuring consistent configuration, simplified onboarding, and alignment with NHS engineering and security standards.

## Prerequisites

### GitHub App setup

Follow these steps to create and configure a minimal‑permission GitHub App that will authenticate the release workflow. This should be done for you by the NHS GitHub Admins. However, you can perform this setup yourself for testing purposes.

1. Create the App

   - Go to [GitHub App settings](https://github.com/settings/apps) user _Settings → Developer Settings → GitHub Apps → New GitHub App_
   - Name it something like _"My Versioning App"_ (must be globally unique)
   - Set the homepage URL to your repository
   - Configure permissions
     - Repository permissions:
       - _Contents: Read & write_, needed to create tags and commit `VERSION`
       - _Issues: Read & write_, enables adding release notes comments
       - _Pull Requests: Read & write_, allows future Pull Request automation and is optional
       - All other repository permissions: No access
     - _Organization permissions: None required_
     - _Account permissions: None required_
   - For the installation scope choose _Only on this account_ (or organisation-wide if required)

2. Generate the private key

   - After saving, click _Generate a private key_
   - Copy the full `.pem` file contents (including BEGIN/END lines)
   - Store it securely, you'll need it to populate `GH_VERSIONING_APP_PRIVATE_KEY`

3. Install the App

   - Click _Install App_ on the App page
   - Choose your user account
   - Select _Only select repositories_ → picking this repository is recommended

4. Add repository variables & secrets

   - Go to your _repository → Settings → Secrets and variables → Actions_
   - Add the variables and secrets listed above in the [Configuration](#configuration) section

5. Test

   - Make a trivial commit e.g. `docs: test app token wiring` to `main`
   - In workflow logs confirm the _"Generate GitHub App token"_ action step succeeds

   After setup, this step in your workflow will issue short-lived authentication tokens automatically:

   ```yaml
   - name: Generate GitHub App token
   uses: actions/create-github-app-token@v2
   with:
       app-id: ${{ vars.GH_VERSIONING_APP_ID }}
       private-key: ${{ secrets.GH_VERSIONING_APP_PRIVATE_KEY }}
   ```

### Bot setup for commit signing

GitHub verifies commits based on user or bot accounts, not apps. This means the release workflow needs a "bot" identity with an associated GPG key.

Steps:

1. Generate a key locally:

   ```bash
   gpg --quick-generate-key "My Signing Bot <your-email@users.noreply.github.com>" ed25519 sign 1m

   ID=XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX # your key ID
   ```

2. Get the **public key**

   ```bash
   gpg --armor --export $ID
   ```

3. Get the **private key**

   ```bash
   gpg --armor --export-secret-keys $ID
   ```

4. Add repository variables & secrets
   - Go to your _repository → Settings → Secrets and variables → Actions_
   - Add the variables and secrets listed above in the [Configuration](#configuration) section

After that, all commits made by the workflow will appear as _"Verified ✅"_ on GitHub.

### Container image signing with Cosign

In addition to commit signing, this repository also demonstrates how to sign and verify container images using Sigstore Cosign. Cosign provides cryptographic assurance that every published image originates from a trusted workflow and has not been altered after build. Each image pushed to the GitHub Container Registry (GHCR) is automatically signed as part of the release workflow. This produces tamper-evident OCI artefacts stored alongside the image, allowing anyone to independently verify its provenance. There are the following benefits:

- End-to-end provenance, extends the trusted chain of custody from commit to container
- Tamper evidence, every signature includes cryptographic metadata that cannot be forged or moved between images
- Transparency, the signature is also recorded in the public Sigstore Rekor transparency log for immutable auditability
- Alignment with NHS Secure by Design, strengthens cyber resilience by ensuring only verified and trusted artefacts reach production

To verify a signed image:

```bash
cosign verify --key cosign.pub ghcr.io/{{ repository }}:{{ version }}
```

The output confirms that:

- the signature matches the published public key,
- the digest corresponds to the exact build produced by the workflow, and
- the signature is recorded in the transparency log if enabled.

This ensures that every deployment can be proven authentic and traceable, a core requirement for secure software supply-chain assurance within NHS.

### Build provenance attestation

To further strengthen software supply-chain assurance, this repository also demonstrates how to generate and publish build provenance attestations using the `actions/attest-build-provenance`
GitHub Action. Provenance describes what was built, how, and by whom - providing cryptographic evidence that each artefact (for example, a container image) was produced by a trusted workflow. When enabled in the workflow, GitHub automatically creates an attestation record linked to the image digest. This record is cryptographically signed using the repository's OpenID Connect (OIDC) identity and stored securely within GitHub's Attestation store. Here are the benefits:

- Verified origin, attests that an artefact was built by a specific repository and workflow run
- Tamper resistance, signed using GitHub's OIDC token, ensuring authenticity and integrity
- Auditable provenance, metadata such as build parameters, commit SHA, and workflow run ID are recorded immutably
- Supply-chain compliance, aligns with [SLSA Level 2+](https://slsa.dev/spec/v1.0/levels) provenance standards

The workflow must request `id-token: write` and `attestations: write` permissions to create attestations. Provenance always references the immutable image digest (sha256:...), not version tags, to ensure traceability. Attestations can be viewed and verified with the GitHub CLI:

```bash
cosign verify-attestation --key cosign.pub ghcr.io/{{ repository }}@sha256:{{ digest }}
```

## Design decisions and rationale

These explanations are meant to help you and me to understand _why_ each part of this setup exists, so none of us has to guess later.

### 🧩 Why use a GitHub App Token instead of a Personal Access Token (PAT)

Using a GitHub App offers significant security and governance advantages:

- Least privilege, the App's installation can be scoped to specific repos with tightly controlled permissions
- Ephemeral credentials, tokens are short-lived and automatically rotated, minimising the impact of leaks
- Easy revocation, uninstalling the App immediately cuts off access, no manual key rotation needed
- Auditability, actions performed by the App are attributed to its installation in GitHub logs
- Operational separation, avoids depending on a human user's PAT, which could carry excessive scopes or expire unexpectedly

In practice, this means safer automation with better traceability and compliance.

Using a single shared bot for an organisation is common practice across the industry. In a large organisation it's standard practice to maintain a single shared bot identity whose purpose is:

- signing commits/tags
- authoring automated release or dependency commits
- interacting via GitHub Apps

This simplifies:

- key management, one GPG keypair to rotate and protect
- auditing, centralised automation identity
- compliance, easier to trace automation actions across repos

Each repository or GitHub App workflow can then reuse that bot's GPG key and email for signing.

### 🔐 Why the signing key belongs to a bot, not the App

GitHub verifies GPG and SSH signatures against user or bot accounts, not apps. Because of that, the release workflow uses a dedicated bot identity that owns the signing key. The benefits are as follows:

- Verified commits and tags, GitHub recognises the signature as trusted and shows the green _"Verified"_ badge
- Separation of authorship, automated releases are distinct from human commits
- Independent key rotation, the key can be rotated or revoked without affecting personal accounts
- Consistent auditing, every release is traceable to a single, identifiable automation user

The public GPG key must be uploaded to the GitHub profile of the account listed in `GIT_SIGNING_BOT_EMAIL`. This allows GitHub to associate the cryptographic signature with that bot's identity and display it as _"Verified"_.

In large organisations, it is entirely appropriate and often preferable to use one shared bot account for signing commits and performing automated releases. This approach reduces key management overhead, simplifies auditing, and provides a single trusted automation identity across all repositories.

> [!NOTE]
> The bot account does not require any repository, workflow or organisation permissions. It acts purely as a cryptographic identity, allowing GitHub to validate signatures and display the _"Verified"_ badge. All actual automation and repository access is performed by the GitHub App or workflow tokens, not by the bot itself.

### 🔏 Why we don't force tag signing by default

Tag signing can be valuable, but it adds complexity in non-interactive CI environments. Here's why it's disabled by default:

- Simplicity, signed annotated tags require a message, without it, Git tries to open an editor and fails in CI
- Reliability, lightweight tags work flawlessly with semantic-release, annotated tags can hang or error if GPG or message handling is misconfigured
- Sufficient provenance, signed commits combined with GitHub Releases already provide a trustworthy audit trail

If you later want to add signed annotated tags, you can do so safely once your workflow is stable, for example:

```bash
git tag -a -m "vX.Y.Z" vX.Y.Z && git push --force origin vX.Y.Z
```

### 📘 Why we don't commit a `CHANGELOG.md`

Instead of maintaining a growing Markdown changelog, this design treats GitHub Releases as the single source of truth. Each release contains its own autogenerated notes, which are easy to view, compare, or query via API. Advantages are:

- Single source of truth, release notes live where users expect them, in the Releases tab
- Cleaner history, release commits only touch the `VERSION` file, avoiding noisy changelog diffs
- Fewer merge conflicts, no simultaneous changelog edits across branches
- Better performance, no rewriting of a large (yes, it will grow with time) markdown file every release
- API-friendly, release data is structured and accessible via GitHub's API for dashboards or audits

If an on-disk changelog is ever needed (e.g. for packaged distributions), re-enable `@semantic-release/changelog` or build an export pipeline that generates one on demand.

### 🐳 Use a flat registry for multiple images

When a single repository produces multiple container images, for example `api` or `ui`, GitHub's Container Registry (GHCR) imposes certain structural and permission constraints on how those images can be stored and tagged. Key points and reasoning:

- GitHub App tokens cannot publish to GHCR, app installation tokens do not carry package-level permissions for container publishing. To push images, use the built-in `${{ github.token }}`, not the legacy `${{ secrets.GITHUB_TOKEN }}`, which automatically grants write access to your repository's package namespace
- Registry scope is flat, the `${{ github.token }}` can only publish to the registry path matching the repository's namespace by default, e.g. `ghcr.io/owner/repo`. Nested namespaces such as `ghcr.io/owner/repo/api` are not permitted when authenticating with default `${{ github.token }}`
- Use tag naming to distinguish components, since subpaths are unavailable, encode both the component name and the version in the image tag. The recommended convention is `<component>-<version>`, for example `ghcr.io/org/repo:api-1.2.3`
- Common in monorepos, this approach works well when multiple related services/components share a single repository and deployment process. However, as the system evolves, it is advisable to separate services into distinct bounded contexts for improved autonomy and flow. See the [Architect for Flow pattern](https://github.com/NHSDigital/software-engineering-quality-framework/blob/main/patterns/architect-for-flow.md) for further guidance.

#### Why this pattern works

- Flat structure, fully compatible with GHCR permissions and the default `${{ github.token }}`
- Established precedent, aligns with common multi-variant image naming, e.g. `nginx:alpine-1.25`, `python:3.12-slim`
- Automation-friendly, uses a shared semantic version across all components
- Discoverable, easy to query, filter, and sort by component prefix `api-*`, `ui-*`, etc.

#### Alternative patterns considered

Other semver-valid formats such as `1.2.3+api` or `api_v1.2.3` were evaluated, but `api-1.2.3` offers the best balance of portability, readability, and compatibility with container tooling and CI/CD pipelines.

#### Detailed configuration

- The repository's package registry entry is created automatically when the first image is published. You should not need to create it manually. Once created, confirm that:
  - The package is explicitly linked to the repository
  - Under _Manage Actions access_, the repository appears in the list and the Role is set to Admin
  - _Inherit access from source repository_ is enabled
  - The package visibility matches the repository's visibility (private or public)
- The GitHub App used for releases does not require Packages access as that capability comes from the ephemeral `${{ github.token }}` used inside the workflow
- However, the workflow itself must request the correct token scopes which must include `packages: write`
- Use `${{ github.token }}` instead of the legacy `${{ secrets.GITHUB_TOKEN }}`, the former is guaranteed to exist in all workflow contexts and is the modern standard
- In _repository → Settings → Actions → General_, ensure the following are configured:
  - _Read repository contents and packages permissions_ under _Workflow permissions_
  - _Allow GitHub Actions to create and approve pull requests_ is optional

This _"flat registry with tagged components"_ model scales cleanly across repositories while remaining compliant with GitHub's authentication and namespace rules. It also provides a consistent, human-readable way to publish and manage multiple container images under one project.

## How to use this repository

### Adding a new feature

1. Create a branch and commit changes using Conventional Commits, for example:

   ```plaintext
   feat(ui): add user authentication
   ```

2. Open a Pull Request and merge it into `main`, ensure that the commit created as an effect of merging this PR contains the above message as this drives the semantic versioning

3. The workflow will:

   - Detect that the change type is `feat` and trigger a minor version bump
   - Update the `VERSION` file
   - Commit and tag the new release
   - Publish release notes automatically

You'll see the new tag and release appear on GitHub, both signed and verified (commit).

### How Conventional Commits affect versioning

| Type              | Example                                  | Version bump | Explanation                                                                                                 |
| ----------------- | ---------------------------------------- | ------------ | ----------------------------------------------------------------------------------------------------------- |
| `docs`            | `docs(readme): update section`           | no bump      | Documentation-only change, does not affect the application's behaviour or API                               |
| `style`           | `style(css): normalise headings`         | no bump      | Code style or formatting change (e.g. whitespace, lint fixes) - no functional impact                        |
| `chore`           | `chore(release): housekeeping`           | no bump      | Maintenance or tooling updates unrelated to user-facing code                                                |
| `test`            | `test(ci): add smoke tests`              | no bump      | Adds or modifies tests, does not change runtime or API behaviour                                            |
| `refactor`        | `refactor(ci): simplify logic`           | patch        | Code improvement or cleanup without changing behaviour, treated like a small fix                            |
| `perf`            | `perf(core): improve runtime`            | patch        | Performance enhancement without altering external behaviour, treated as a fix                               |
| `fix`             | `fix(ci): correct signing config`        | patch        | Corrects an existing issue, triggers a patch version bump (`x.y.z → x.y.(z+1)`)                             |
| `feat`            | `feat(ci): add exec plugin`              | minor        | Introduces a new, backward-compatible feature, triggers a minor version bump (`x.y.z → x.(y+1).0`)          |
| `<type>[scope]!:` | `feat(api)!: remove deprecated endpoint` | major        | Introduces a breaking change (non-backward-compatible), triggers a major version bump (`x.y.z → (x+1).0.0`) |

### Final thought

This repository isn't just a demo, it's a living reference for how NHS teams can manage automated versioning and release engineering properly:

- Secure by default
- Fully automated
- Transparent and traceable
- Easy for newcomers to understand

If you're reading this and thinking _"I'm not sure I understand all of it"_, that's ok. Just clone it, run it, and watch what happens. Each step is logged, readable, and reversible. The goal is not perfection, it's confidence!

## Outstanding

- [ ] Validate that the workflow functions correctly for private repositories
- [ ] Explore manually created nested registry packages connected to the repository, for example `ghcr.io/owner/repo/api:0.0.1`. However, this is only a nice-to-have, the preferred approach is to decompose the monorepo to support product and team autonomy when delivering services at a national scale. This is the alignment with modern DevOps and Conway's Law, promoting smaller, autonomous repositories that map to products and teams is the scalable, maintainable path for large, national services
