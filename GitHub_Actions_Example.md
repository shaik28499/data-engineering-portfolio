# Sample GitHub Actions Workflow for ETL Pipeline

```yaml
# .github/workflows/etl-pipeline.yml
name: ETL Pipeline Deployment

on:
  push:
    branches: 
      - main
      - develop
    paths-ignore:
      - README.md
      - docs/**
      
  pull_request:
    branches: 
      - main
    
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        required: true
        default: 'dev'
        type: choice
        options:
        - dev
        - test
        - prod

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  validate:
    name: 'Validate Infrastructure'
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.5.0
          terraform_wrapper: false
          
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        
      - name: Terraform Init
        run: terraform init -backend=false
        
      - name: Terraform Validate
        run: terraform validate

  plan:
    name: 'Plan Infrastructure'
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'
    
    outputs:
      tfplanExitCode: ${{ steps.tf-plan.outputs.exitcode }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.5.0
          terraform_wrapper: false
          
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          
      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="bucket=${{ vars.TERRAFORM_STATE_BUCKET }}" \
            -backend-config="key=etl-pipeline/${{ vars.ENVIRONMENT }}/terraform.tfstate" \
            -backend-config="region=${{ vars.AWS_REGION }}"
            
      - name: Terraform Plan
        id: tf-plan
        run: |
          export exitcode=0
          terraform plan \
            -var-file=environments/${{ vars.ENVIRONMENT }}.tfvars \
            -detailed-exitcode \
            -no-color \
            -out tfplan || export exitcode=$?
          
          echo "exitcode=$exitcode" >> $GITHUB_OUTPUT
          
          if [ $exitcode -eq 1 ]; then
            echo "Terraform Plan Failed!"
            exit 1
          else 
            exit 0
          fi
          
      - name: Publish Terraform Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ vars.ENVIRONMENT }}
          path: tfplan

  deploy:
    name: 'Deploy Infrastructure'
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: ${{ vars.ENVIRONMENT }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.5.0
          terraform_wrapper: false
          
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          
      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="bucket=${{ vars.TERRAFORM_STATE_BUCKET }}" \
            -backend-config="key=etl-pipeline/${{ vars.ENVIRONMENT }}/terraform.tfstate" \
            -backend-config="region=${{ vars.AWS_REGION }}"
            
      - name: Terraform Plan
        run: |
          terraform plan \
            -var-file=environments/${{ vars.ENVIRONMENT }}.tfvars \
            -out tfplan
            
      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
        
      - name: Upload ETL Scripts
        run: |
          aws s3 sync ./scripts/ s3://${{ vars.ETL_BUCKET }}/scripts/ \
            --exclude "*.pyc" \
            --exclude "__pycache__/*"
            
      - name: Notify Success
        if: success()
        run: |
          echo "✅ ETL Pipeline deployed successfully to ${{ vars.ENVIRONMENT }}"
          
      - name: Notify Failure
        if: failure()
        run: |
          echo "❌ ETL Pipeline deployment failed for ${{ vars.ENVIRONMENT }}"

  test:
    name: 'Test ETL Jobs'
    runs-on: ubuntu-latest
    needs: deploy
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-region: ${{ vars.AWS_REGION }}
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          
      - name: Run ETL Job Test
        run: |
          # Start a test Glue job
          JOB_RUN_ID=$(aws glue start-job-run \
            --job-name "${{ vars.ENVIRONMENT }}-test-etl-job" \
            --arguments '{"--test-mode":"true"}' \
            --query 'JobRunId' \
            --output text)
          
          echo "Started test job run: $JOB_RUN_ID"
          
          # Wait for job completion (simplified)
          sleep 300
          
          # Check job status
          JOB_STATUS=$(aws glue get-job-run \
            --job-name "${{ vars.ENVIRONMENT }}-test-etl-job" \
            --run-id "$JOB_RUN_ID" \
            --query 'JobRun.JobRunState' \
            --output text)
          
          if [ "$JOB_STATUS" = "SUCCEEDED" ]; then
            echo "✅ ETL job test passed"
          else
            echo "❌ ETL job test failed with status: $JOB_STATUS"
            exit 1
          fi
```

## Environment Variables Required

```yaml
# Repository Variables (Settings > Secrets and variables > Actions > Variables)
AWS_REGION: us-east-1
AWS_ROLE_ARN: arn:aws:iam::123456789012:role/github-actions-role
TERRAFORM_STATE_BUCKET: my-terraform-state-bucket
ENVIRONMENT: dev
ETL_BUCKET: my-etl-scripts-bucket
```

## Secrets Required

```yaml
# Repository Secrets (Settings > Secrets and variables > Actions > Secrets)
# None required if using OIDC with IAM roles
# Otherwise add:
# AWS_ACCESS_KEY_ID: your-access-key
# AWS_SECRET_ACCESS_KEY: your-secret-key
```

## Workflow Features

- **Multi-environment support**: dev, test, prod
- **Terraform validation**: Format, init, validate
- **Plan on PR**: Shows infrastructure changes
- **Deploy on merge**: Automatic deployment to main
- **ETL job testing**: Validates deployed jobs
- **Error handling**: Proper failure notifications
- **Security**: Uses OIDC for AWS authentication