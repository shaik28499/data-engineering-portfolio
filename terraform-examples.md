# Sample Terraform Configuration for AWS Glue ETL Pipeline

## Glue Job Resource Example

```hcl
# examples/terraform/glue-job.tf
resource "aws_glue_job" "etl_job" {
  name         = "${var.environment}-${var.job_name}"
  role_arn     = var.glue_service_role_arn
  glue_version = "5.0"
  
  command {
    name            = "glueetl"
    script_location = "s3://${var.scripts_bucket}/scripts/${var.script_name}"
    python_version  = "3"
  }
  
  default_arguments = {
    "--job-bookmark-option"              = "job-bookmark-enable"
    "--enable-metrics"                   = "true"
    "--enable-continuous-cloudwatch-log" = "true"
    "--enable-spark-ui"                  = "true"
    "--spark-event-logs-path"            = "s3://${var.logs_bucket}/spark-logs/"
    "--TempDir"                          = "s3://${var.temp_bucket}/temp/"
  }
  
  execution_property {
    max_concurrent_runs = var.max_concurrent_runs
  }
  
  worker_type       = var.worker_type
  number_of_workers = var.number_of_workers
  timeout           = var.timeout_minutes
  max_retries       = var.max_retries
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-${var.job_name}"
      Type = "ETL"
    }
  )
}

# Variables
variable "environment" {
  description = "Environment name (dev, test, prod)"
  type        = string
}

variable "job_name" {
  description = "Name of the Glue job"
  type        = string
}

variable "glue_service_role_arn" {
  description = "ARN of the IAM role for Glue service"
  type        = string
}

variable "worker_type" {
  description = "Type of predefined worker (G.1X, G.2X, etc.)"
  type        = string
  default     = "G.1X"
}

variable "number_of_workers" {
  description = "Number of workers"
  type        = number
  default     = 5
}
```

## Glue Connection Example

```hcl
# examples/terraform/glue-connection.tf
resource "aws_glue_connection" "database_connection" {
  name = "${var.environment}-${var.connection_name}"
  
  connection_properties = {
    JDBC_CONNECTION_URL = var.jdbc_connection_url
    USERNAME           = var.db_username
    PASSWORD           = var.db_password
    JDBC_ENFORCE_SSL   = "false"
  }
  
  physical_connection_requirements {
    availability_zone      = var.availability_zone
    security_group_id_list = var.security_group_ids
    subnet_id             = var.subnet_id
  }
  
  tags = var.common_tags
}
```

## S3 Bucket for ETL Artifacts

```hcl
# examples/terraform/s3-bucket.tf
resource "aws_s3_bucket" "etl_bucket" {
  bucket = "${var.environment}-${var.project_name}-etl-artifacts"
  
  tags = merge(
    var.common_tags,
    {
      Purpose = "ETL Scripts and Artifacts"
    }
  )
}

resource "aws_s3_bucket_versioning" "etl_bucket_versioning" {
  bucket = aws_s3_bucket.etl_bucket.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "etl_bucket_encryption" {
  bucket = aws_s3_bucket.etl_bucket.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```