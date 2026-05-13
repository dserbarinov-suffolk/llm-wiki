# CI/CD Requirements

> Requirements for CI/CD pipeline. Actual GitHub Actions workflows live in `.github/workflows/`.

---

## Required Jobs

| Job               | Trigger            | Purpose                      |
| ----------------- | ------------------ | ---------------------------- |
| lint-and-test     | All PRs and pushes | Code quality gates           |
| build-artifacts   | All PRs and pushes | Build and validate artifacts |
| deploy-foundation | Push to env branch | Deploy foundation stack      |
| deploy-app        | Push to env branch | Deploy app stack             |
| smoke-test        | After deploy       | Verify deployment health     |
| drift-check       | Daily schedule     | Detect configuration drift   |

---

## Quality Gates

### PR Requirements

Before merge, PRs must pass:

1. **Lint**: ESLint, Prettier, Python flake8/black
2. **TypeScript**: `tsc --noEmit` passes
3. **Unit tests**: All tests pass
4. **ASL validation**: Step Functions definition is valid
5. **CloudFormation lint**: cfn-lint passes
6. **Artifact build**: All artifacts build successfully

### Deployment Gates

Before deployment:

1. All quality gates pass
2. Change set created successfully
3. Change set contains no dangerous changes (see below)
4. Previous environment deployed (dev → stage → prod)

---

## ASL Validation

The Step Functions state machine definition must be validated before deployment:

```bash
aws stepfunctions validate-state-machine-definition \
  --definition "$(cat dist/step-functions/processing-workflow.asl.json)" \
  --type STANDARD
```

Validation must pass with no errors. Warnings are acceptable.

---

## CloudFormation Lint

All CloudFormation templates must pass cfn-lint:

```bash
cfn-lint infra/cloudformation/**/*.yaml
```

No errors allowed. Warnings should be reviewed but may be acceptable.

---

## Artifact Build Requirements

### Build Commands

```bash
# Full build (CI)
pnpm build:artifacts

# Individual builds
pnpm build:lambda:node
pnpm build:lambda:python
pnpm build:layers
pnpm build:step-functions
```

### Build Verification

After build, verify:

1. All expected artifacts exist in `dist/`
2. Artifacts are within size limits (Lambda: 50MB, layers: 250MB)
3. ASL passes validation

---

## Change Set Safety Checks

Before executing a CloudFormation change set, verify no dangerous changes:

```bash
CHANGES=$(aws cloudformation describe-change-set \
  --stack-name "$STACK_NAME" \
  --change-set-name "$CHANGE_SET_NAME" \
  --query 'Changes[?ResourceChange.Replacement==`True` || ResourceChange.Action==`Delete`]')

if [[ "$CHANGES" != "[]" ]]; then
  echo "Change set contains DELETE or REPLACE operations"
  exit 1
fi
```

Dangerous changes that require manual approval:

- DELETE on any resource
- REPLACE on retained resources (buckets, tables, secrets)

---

## Environment Promotion Rules

| From  | To    | Requirements                                 |
| ----- | ----- | -------------------------------------------- |
| -     | dev   | Quality gates pass                           |
| dev   | stage | Quality gates pass, dev deployed             |
| stage | prod  | Quality gates pass, stage deployed, approval |

### Approval Requirements

Production deployments require:

1. GitHub Environment protection rule
2. At least one approver
3. Fresh approval (within 24 hours)

---

## Smoke Test Requirements

After deployment, smoke tests must verify:

1. **API health**: GET /health returns 200
2. **DynamoDB access**: Can read/write to ProcessingJobs
3. **S3 access**: Can read/write to Processing bucket
4. **Secrets access**: Can read AppEnvSecret
5. **Queue access**: Can send to intake queue

### Smoke Test Timeout

Smoke tests must complete within 5 minutes. Timeout triggers rollback.

---

## Artifact Upload

Artifacts must be uploaded before stack deployment:

```bash
VERSION="${GITHUB_SHA}"
BUCKET=$(aws cloudformation describe-stacks \
  --stack-name "processing-foundation-${ENV}" \
  --query 'Stacks[0].Outputs[?OutputKey==`DeploymentArtifactsBucketName`].OutputValue' \
  --output text)

# Lambda functions
for f in dist/lambda/node/*.zip dist/lambda/python/*.zip; do
  aws s3 cp "$f" "s3://${BUCKET}/lambda/${VERSION}/$(basename $f)"
done

# Layers
for f in dist/layers/*.zip; do
  aws s3 cp "$f" "s3://${BUCKET}/layers/${VERSION}/$(basename $f)"
done

# Step Functions
aws s3 cp dist/step-functions/processing-workflow.asl.json \
  "s3://${BUCKET}/sfn/${VERSION}/workflow.asl.json"
```

---

## Secrets Management

### Secrets in CI

| Secret             | Scope       | Purpose                  |
| ------------------ | ----------- | ------------------------ |
| AWS_ROLE_TO_ASSUME | Environment | OIDC role for AWS access |

### OIDC Configuration

Use GitHub OIDC for AWS authentication (no long-lived credentials):

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME }}
    aws-region: us-east-1
```

---

## Concurrency Control

Prevent concurrent deployments to the same environment:

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: false
```

---

## Notification Requirements

### Success Notifications

- Slack message to #deployments channel
- Include: environment, version, duration

### Failure Notifications

- Slack message to #deployments channel (urgent)
- Include: environment, version, failure reason, link to logs

---

## Changelog

| Date       | Change                         |
| ---------- | ------------------------------ |
| 2026-05-13 | Initial requirements extracted |
