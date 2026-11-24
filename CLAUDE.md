# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a **CloudFormation template repository** that provides a reusable nested stack template for deploying S3 buckets with security best practices. The template is designed to be referenced by parent CloudFormation stacks via `TemplateURL`.

**Key characteristics:**

- One nested template: `cloudformation/template.yaml`
- Parameterized bucket naming with project, account ID, environment, and region
- S3 security defaults: versioning enabled by default (configurable), all four public access blocks enabled
- Optional SSE-KMS encryption (key name, alias, or ARN), with optional Bucket Key
- Optional access logging to a dedicated, locked-down logging bucket
- Optional static website hosting
- Optional S3 event notifications to up to 5 SQS queues, each with prefix and suffix filters
- Automated semantic versioning and releases from `main`
- AWS OIDC authentication for CI deployments

## Project Structure

```text
cloudformation/
├── template.yaml                 # Nested template: S3 bucket (+ optional logging bucket)
├── parameters.json               # Parameter values used by CI and manual deploys
└── stack-config.json             # CI config: stackName, templateFile, parameterFile

.env/
└── environments.yaml             # GitHub environments (ci, devl) and target regions

.github/workflows/
├── ci.yaml                       # Calls the reusable CI build workflow for the template
├── release.yaml                  # semantic-release on push to main
├── setup-environments.yaml       # Sets up AWS environments (reusable workflow)
├── create-branch.yaml            # Auto-creates feature branches from assigned issues
├── notify.yaml                   # Slack notification on PR/issue opened
├── claude.yaml                   # @claude mentions on issues and PRs
└── claude-code-review.yaml       # Automated Claude code review

scripts/plugins/
├── release.config.js             # Shared semantic-release configuration
├── analyze-commits.js            # Commit analysis for release type
├── generate-notes.js             # Release notes generation
├── prepare.js / publish.js       # semantic-release lifecycle hooks
└── verify-conditions.js          # Pre-release checks

.releaserc.json                   # semantic-release plugins (active config for main)
CHANGELOG.md                      # Maintained by semantic-release
package.json                      # Dependencies: semantic-release, commitizen
README.md                         # Template documentation and usage examples
.claude/.skills/                  # Project skills for Claude Code
```

## Development Commands

### Install dependencies

```bash
npm ci
```

### Commit with conventional commit format

```bash
npx cz commit
```

Select `feat`, `fix`, or `chore` (or `docs`, etc.). Only `feat`, `fix`, and breaking changes trigger a release.

### Trigger semantic release (normally automatic on main)

```bash
npm run release
```

### Validate the template locally

```bash
cfn-lint cloudformation/template.yaml
aws cloudformation validate-template --template-body file://cloudformation/template.yaml --region us-east-1
```

## Template Architecture

### Nested Stack Pattern

The template is **nested** — it is referenced from a parent stack through `AWS::CloudFormation::Stack` with `TemplateURL`. It exports outputs for the parent to consume.

- **Parent stack** calls `AWS::CloudFormation::Stack` with `TemplateURL` pointing to the uploaded template in S3
- **Nested template** exposes `S3BucketName`, `S3BucketArn`, and (when website hosting is on) `S3BucketWebsiteEndpoint`
- **Parent** reads them with `!GetAtt NestedStack.Outputs.OutputKey`

### Bucket Naming Convention

```bash
{ProjectName}-{BucketBaseName}-{AccountId}-{Environment}-{Region}[-{CiSuffix}]
```

The logging bucket uses the same pattern with `logs` inserted after `BucketBaseName`.

Example: `cfn-nested-s3-bucket-123456789012-devl-us-east-1`

The name must be unique per account and region, isolate environments, and be predictable for automation.

### Parameters

| Group | Parameters |
| ----- | ---------- |
| S3 bucket | `ProjectName`, `BucketBaseName` |
| Encryption | `KmsKey`, `EnableBucketKey` |
| Event notifications | `QueueArn`, `QueueEvent`, `QueueFilterPrefix`, `QueueFilterSuffix` (slot 1), plus `…2`–`…5` suffixed variants for slots 2–5 |
| Advanced | `Environment` (`devl`, `test`, `prod`), `CiSuffix`, `WebsiteConfiguration`, `EnableLogging`, `EnableVersioning` |

Behavior that is easy to get wrong:

- `KmsKey` empty → SSE-S3 (`AES256`). Otherwise SSE-KMS. Plain names are converted to `alias/<name>`.
- `EnableBucketKey` only takes effect with a KMS key.
- `QueueArn` (and `QueueArn2`–`QueueArn5`) empty → that slot is skipped. If all are empty, there is no `NotificationConfiguration`. Each set slot adds one `QueueConfigurations` entry with its event type. `Filter` appears only when a prefix or suffix is set on that slot.
- A queue used for notifications must be a standard SQS queue (not FIFO), and its policy must allow `s3.amazonaws.com` to send messages from this bucket. S3 checks this when the notification configuration is applied, so the queue policy must exist before the stack deploys.
- Access logging (`EnableLogging=true`) creates `LoggingBucket` with the same public access blocks.

### Security Controls

- Versioning on the primary bucket is controlled by `EnableVersioning` (default `true`). When `false`, `VersioningConfiguration` is omitted (not `Suspended`).
- All four public access block settings on both buckets
- `Metadata` → `com.aws.cloudformation.Context` documents the intent and must-rules of each bucket. Keep these in sync when behavior changes.

## Key Files

### `cloudformation/template.yaml`

The only template. Changes here are what the CI workflow validates and deploys.

### `cloudformation/parameters.json`

Parameter values used by the CI workflow and as a reference for manual deploys. Keys must match parameter names in the template. Current values set `WebsiteConfiguration` and `EnableBucketKey` to `true` and leave notifications disabled (no `QueueArn`).

### `cloudformation/stack-config.json`

Read by the CI `load-config` job. It sets `stackName`, `templateFile`, and `parameterFile`. Update it if the template or parameter file moves.

### `.github/workflows/ci.yaml`

**Triggered on:**

- `workflow_dispatch` (manual)
- Pushes to `feature/**` and `bug/**` branches, when `cloudformation/**` or `ci.yaml` changes
- Pull requests to `main`, when `cloudformation/**` changes

**Process:**

1. `load-config` reads `cloudformation/stack-config.json` and writes a step summary
2. `ci-build` calls the reusable workflow `subhamay-bhattacharyya-gha/cfn-ci-build-reusable-wf` with `environment: ci`, the stack name, the template file, and the parameter file. That workflow:
   - Generates a random CI suffix, so each run gets its own stack name (`{stackName}-{suffix}`)
   - Deploys the stack with the cfn-deploy reusable workflow
   - Deletes the stack afterwards
3. `generate-changelog` and `create-release` run only on `main`

The reusable workflow accepts only file paths. It does not accept parameter overrides, so any value that must differ in CI has to live in `cloudformation/parameters.json` in this repo.

### `.github/workflows/release.yaml`

Runs `npx semantic-release` on every push to `main`, using Node 22.14.0 and `npm ci`. Config is in `.releaserc.json`, which uses the commit analyzer, release notes generator, changelog, git, and GitHub plugins.

**Release rules:**

- `feat:` → MINOR bump (0.1.0 → 0.2.0)
- `fix:` → PATCH bump (0.1.0 → 0.1.1)
- `BREAKING CHANGE:` footer → MAJOR bump (0.1.0 → 1.0.0)
- Other types (`chore`, `docs`, etc.) → no release

Note: `ci.yaml`'s `generate-changelog` job calls `orhun/git-cliff-action` with `cliff.toml`, which does not exist in the repo. That job would fail as written. Also, `CHANGELOG.md` is written by semantic-release, so these two jobs overlap.

## GitHub Environments & Credentials

- Environments are defined in `.env/environments.yaml`: `ci` and `devl` both map to `AWS-CFN-TEMPLATES`, with region `us-east-1`
- The reusable workflows read these repository/environment variables: `AWS_ENVIRONMENT`, `AWS_REGION`, `AWS_ACCOUNT_ID`, `AWS_OIDC_ROLE_NAME`, `CFN_TEMPLATE_S3_BUCKET_NAME`
- Authentication is keyless AWS OIDC. The GitHub OIDC provider in the target account must trust the role named by `AWS_OIDC_ROLE_NAME`.
- `setup-environments.yaml` provisions these environments and uses `secrets.GH_PAT`

Never commit account IDs, role ARNs, or credentials into the repo beyond what `parameters.json` already holds.

## Testing & Validation

Before opening a PR for a template change:

1. `cfn-lint cloudformation/template.yaml` must pass
2. `aws cloudformation validate-template` must succeed
3. If you added or renamed a parameter, update `cloudformation/parameters.json` (if the default should differ in CI) and the parameter table in `README.md`
4. If you changed bucket behavior, update the `Metadata` context on the affected resource and the README feature list

CI runs the deploy-and-delete cycle automatically on PRs and feature pushes.

## Conventional Commits & Release Flow

Commit types drive semantic versioning. Use `npx cz commit` or write the message by hand in the same format:

- `feat: add support for X` → MINOR release
- `fix: correct behavior of Y` → PATCH release
- `chore: update deps` → no release
- `docs: clarify README` → no release

Only commits that land on `main` trigger releases. Feature branches use the same format; the release happens when the PR merges.

## When Modifying the Template

1. **Edit** `cloudformation/template.yaml`
2. **Update** `cloudformation/parameters.json` if the CI values need to change
3. **Validate** with `cfn-lint` and `aws cloudformation validate-template`
4. **Update** `README.md` (parameter table, options, notes) and this file if behavior changed
5. **Commit** with a conventional message, e.g. `feat: add SQS event notifications`
6. **Open a PR** to `main`. CI deploys and deletes a temporary stack
7. **Merge to `main`**. The release workflow creates the version tag, changelog entry, and GitHub release

## Branches

`main` is the release branch. Feature work branches from `main` and merges back through a PR. Branch names follow `{type}/CFN-{issue-number}-{slug}`, e.g. `feature/CFN-42-add-encryption`. The `create-branch.yaml` workflow creates these from assigned issues, using the `CFN` prefix.
