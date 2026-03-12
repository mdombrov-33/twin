# AI Digital Twin

Learning project - building and deploying a personal AI chatbot that represents you on your website. Covers the full stack: local development, manual AWS setup, Infrastructure as Code with Terraform, and CI/CD with GitHub Actions.

## Stack

- **Frontend**: Next.js (App Router) + TypeScript + Tailwind CSS
- **Backend**: Python FastAPI + Mangum (Lambda adapter)
- **AI**: OpenAI GPT-4o-mini → AWS Bedrock (Nova models)
- **Storage**: File-based memory → S3 (conversation history as JSON)
- **Infra**: AWS Lambda + API Gateway + S3 + CloudFront
- **IaC**: Terraform with workspaces (dev/test/prod)
- **CI/CD**: GitHub Actions + OIDC (no long-lived keys)

## Project Structure

```
twin/
├── backend/          # FastAPI app + Lambda handler
│   ├── server.py
│   ├── lambda_handler.py
│   ├── context.py    # builds system prompt
│   ├── resources.py  # loads personal data files
│   ├── deploy.py     # builds lambda-deployment.zip via Docker
│   └── data/         # facts.json, summary.txt, style.txt, linkedin.pdf
├── frontend/         # Next.js app
│   ├── app/page.tsx
│   └── components/twin.tsx
├── memory/           # local conversation JSON files (gitignored)
├── scripts/
│   ├── deploy.sh / deploy.ps1
│   └── destroy.sh / destroy.ps1
└── terraform/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── versions.tf
    ├── backend.tf
    └── terraform.tfvars
```

---

## Local Development

### Backend

```bash
# Install uv (fast Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

cd backend
uv init --bare
uv python pin 3.14
uv add -r requirements.txt
uv run uvicorn server:app --reload
# → http://localhost:8000
```

`backend/.env`:

```
OPENAI_API_KEY=sk-...
CORS_ORIGINS=http://localhost:3000
```

### Frontend

```bash
# Create Next.js app (first time only)
npx create-next-app@latest frontend --typescript --tailwind --app --no-src-dir
cd frontend
npm install lucide-react
npm run dev
# → http://localhost:3000
```

`frontend/.env.local` (for local dev):

```
NEXT_PUBLIC_API_URL=http://localhost:8000
```

---

## Manual AWS Setup (Console)

Progression done before Terraform — understanding each service manually.

### IAM Group `TwinAccess`

Policies attached:

- `AWSLambda_FullAccess`
- `AmazonS3FullAccess`
- `AmazonAPIGatewayAdministrator`
- `CloudFrontFullAccess`
- `IAMReadOnlyAccess`
- `AmazonBedrockFullAccess`
- `CloudWatchFullAccess`
- `AmazonDynamoDBFullAccess`

### Lambda Function `twin-api`

- Runtime: Python 3.14 / x86_64
- Handler: `lambda_handler.handler`
- Timeout: 30s+
- Env vars: `OPENAI_API_KEY`, `CORS_ORIGINS`, `USE_S3=true`, `S3_BUCKET`, `BEDROCK_MODEL_ID`
- Attach policy: `AmazonS3FullAccess`, `AmazonBedrockFullAccess`

### Build & upload Lambda package

```bash
# Requires Docker running (builds for linux/amd64)
cd backend
uv run deploy.py
# → creates lambda-deployment.zip

# Upload via S3 (more reliable than direct upload)
DEPLOY_BUCKET="twin-deploy-$(date +%s)"
aws s3 mb s3://$DEPLOY_BUCKET
aws s3 cp lambda-deployment.zip s3://$DEPLOY_BUCKET/
aws lambda update-function-code \
    --function-name twin-api \
    --s3-bucket $DEPLOY_BUCKET \
    --s3-key lambda-deployment.zip \
    --region eu-central-1

# Cleanup
aws s3 rm s3://$DEPLOY_BUCKET/lambda-deployment.zip
aws s3 rb s3://$DEPLOY_BUCKET
```

### S3 Buckets

```bash
# Memory bucket (private, blocks public access)
aws s3 mb s3://twin-memory-XXXX

# Frontend bucket (public static website)
aws s3 mb s3://twin-frontend-XXXX
# Enable static website hosting via console: index.html / 404.html
# Add bucket policy: s3:GetObject on arn:aws:s3:::twin-frontend-XXXX/*
```

### API Gateway (HTTP API)

- Type: HTTP API
- Routes: `GET /`, `GET /health`, `POST /chat`, `ANY /{proxy+}`
- Integration: Lambda proxy → `twin-api`
- CORS: `allow_origins=*`, `allow_headers=*`, `allow_methods=*`

```bash
# Test
curl https://XXXXXX.execute-api.eu-central-1.amazonaws.com/health
```

### CloudFront

- Origin: S3 static website endpoint (HTTP only — S3 website doesn't support HTTPS)
- Default root object: `index.html`
- Viewer protocol: Redirect HTTP to HTTPS
- `CORS_ORIGINS` Lambda env var must match exactly: `https://dXXXXXX.cloudfront.net` (no trailing slash)

### Deploy Frontend

```bash
cd frontend
# Set production API URL
echo "NEXT_PUBLIC_API_URL=https://XXXXXX.execute-api.eu-central-1.amazonaws.com" > .env.production
npm run build
aws s3 sync out/ s3://twin-frontend-XXXX/ --delete

# Invalidate CloudFront cache after update
aws cloudfront create-invalidation --distribution-id EXXXXXX --paths "/*"
```

---

## Switch to AWS Bedrock

Replace OpenAI with Bedrock in `server.py`:

```bash
# Remove openai from requirements.txt, keep boto3
cd backend
uv add -r requirements.txt
```

Lambda env var:

```
BEDROCK_MODEL_ID=global.amazon.nova-2-lite-v1:0
DEFAULT_AWS_REGION=eu-central-1
```

Model IDs (use `global.` prefix for cross-region inference — higher quotas):

- `global.amazon.nova-2-micro-v1:0` — fastest/cheapest
- `global.amazon.nova-2-lite-v1:0` — balanced (default)
- `global.amazon.nova-2-pro-v1:0` — most capable

If quota issues, try regional prefixes: `us.`, `eu.`, `ap.`

---

## Terraform (IaC)

Manages everything: S3 buckets, Lambda, API Gateway, CloudFront, IAM roles. Uses workspaces for environment isolation.

### Install

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform --version
```

### Init & deploy

```bash
cd terraform
terraform init
terraform workspace new dev   # or: terraform workspace select dev
terraform plan
terraform apply -var="project_name=twin" -var="environment=dev" -auto-approve
```

### Key files

**`terraform/versions.tf`** — provider config (AWS ~6.0)

**`terraform/variables.tf`** — `project_name`, `environment`, `bedrock_model_id`, `lambda_timeout`, throttle limits, `use_custom_domain`, `root_domain`

**`terraform/terraform.tfvars`**:

```hcl
project_name             = "twin"
environment              = "dev"
bedrock_model_id         = "global.amazon.nova-2-micro-v1:0"
lambda_timeout           = 60
api_throttle_burst_limit = 10
api_throttle_rate_limit  = 5
use_custom_domain        = false
root_domain              = ""
```

**`terraform/prod.tfvars`** (optional, for custom domain):

```hcl
use_custom_domain = true
root_domain       = "yourdomain.com"
bedrock_model_id  = "global.amazon.nova-2-lite-v1:0"
```

**Resource naming**: `twin-dev-api`, `twin-dev-memory-ACCOUNTID`, etc.

### Workspaces

```bash
terraform workspace list
terraform workspace select dev
terraform workspace show
```

### Destroy

```bash
# Empty S3 first (Terraform can't delete non-empty buckets)
aws s3 rm s3://twin-dev-frontend-ACCOUNTID --recursive
aws s3 rm s3://twin-dev-memory-ACCOUNTID --recursive

terraform destroy -var="project_name=twin" -var="environment=dev" -auto-approve

# Remove workspace
terraform workspace select default
terraform workspace delete dev
```

---

## Deployment Scripts

One-command deploy for any environment:

```bash
# Mac/Linux
chmod +x scripts/deploy.sh scripts/destroy.sh

./scripts/deploy.sh dev     # builds Lambda, runs Terraform, builds+syncs frontend
./scripts/deploy.sh test
./scripts/deploy.sh prod    # uses prod.tfvars

./scripts/destroy.sh dev    # empties S3, runs terraform destroy
```

```powershell
# Windows
.\scripts\deploy.ps1 -Environment dev
.\scripts\destroy.ps1 -Environment dev
```

The deploy script:

1. Runs `uv run deploy.py` → builds `lambda-deployment.zip` via Docker
2. `terraform init` with S3 backend
3. `terraform workspace select/new $ENV`
4. `terraform apply`
5. Writes `frontend/.env.production` with API URL from Terraform output
6. `npm run build` + `aws s3 sync`

---

## Terraform Remote State (S3 Backend)

Needed for CI/CD — state stored in S3 with DynamoDB locking.

### Create state bucket & lock table (one-time)

```bash
cd terraform
terraform workspace select default
terraform init

# Apply only backend resources
terraform apply \
  -target=aws_s3_bucket.terraform_state \
  -target=aws_s3_bucket_versioning.terraform_state \
  -target=aws_s3_bucket_server_side_encryption_configuration.terraform_state \
  -target=aws_s3_bucket_public_access_block.terraform_state \
  -target=aws_dynamodb_table.terraform_locks

# Remove the setup file after
rm terraform/backend-setup.tf
```

**`terraform/backend.tf`**:

```hcl
terraform {
  backend "s3" {
    # configured at runtime via -backend-config flags
  }
}
```

Init with backend:

```bash
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
terraform init \
  -backend-config="bucket=twin-terraform-state-${AWS_ACCOUNT_ID}" \
  -backend-config="key=dev/terraform.tfstate" \
  -backend-config="region=eu-central-1" \
  -backend-config="dynamodb_table=twin-terraform-locks" \
  -backend-config="encrypt=true"
```

---

## CI/CD — GitHub Actions

### AWS OIDC (no long-lived keys)

Creates an IAM role that GitHub Actions can assume via OIDC — no stored AWS credentials.

```bash
cd terraform
terraform workspace select default
terraform apply -target=aws_iam_openid_connect_provider.github \
                -target=aws_iam_role.github_actions \
                -var="github_repository=YOUR_USERNAME/YOUR_REPO"

# Note the output: github_actions_role_arn
```

### GitHub Secrets

Set in repo → Settings → Secrets and variables → Actions:

| Secret           | Value                                                    |
| ---------------- | -------------------------------------------------------- |
| `AWS_ROLE_ARN`   | `arn:aws:iam::ACCOUNTID:role/github-actions-twin-deploy` |
| `AWS_REGION`     | `eu-central-1`                                           |
| `AWS_ACCOUNT_ID` | your 12-digit account ID                                 |

### Workflows

**`.github/workflows/deploy.yml`** — triggers on push to `main`, deploys to `dev`. Manual trigger supports `test`/`prod`.

**`.github/workflows/destroy.yml`** — manual trigger, destroys any environment.

Key workflow pattern:

```yaml
permissions:
  id-token: write   # needed for OIDC
  contents: read

- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: ${{ secrets.AWS_REGION }}
```

---

## Useful AWS CLI Commands

```bash
# Check Lambda status
aws lambda get-function --function-name twin-dev-api

# View CloudWatch logs
aws logs tail /aws/lambda/twin-dev-api --follow

# List S3 buckets
aws s3 ls | grep twin

# CloudFront invalidation
aws cloudfront list-distributions --query "DistributionList.Items[].{id:Id,domain:DomainName}"
aws cloudfront create-invalidation --distribution-id EXXXXXX --paths "/*"

# Get API Gateway URL
aws apigatewayv2 get-apis --query "Items[?Name=='twin-dev-api-gateway'].ApiEndpoint"

# Terraform state inspection
terraform show
terraform state list
terraform output
```
