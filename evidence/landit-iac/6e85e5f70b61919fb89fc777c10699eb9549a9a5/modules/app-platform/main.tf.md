# main.tf — 공개 근거 발췌

- 원본 저장소: `landit-iac` (비공개).
- 원본 커밋: `6e85e5f70b61919fb89fc777c10699eb9549a9a5`.
- 원본 경로: `modules/app-platform/main.tf`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–330행


````hcl
# ECS Fargate 기반 Landit application platform 리소스를 정의한다.
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_caller_identity" "current" {}

locals {
  name_prefix     = "${var.environment}-${var.project_name}"
  ssm_path        = trimsuffix(var.parameter_store_path, "/")
  app_bucket_name = "${local.name_prefix}-app-${data.aws_caller_identity.current.account_id}"
  enable_https    = var.ecs_platform_enabled && var.alb_certificate_arn != null
  enable_api_host_rule = (
    local.enable_https &&
    length(var.api_domain_name) > 0
  )
  enable_ai_host_rule = (
    local.enable_https &&
    length(var.ai_domain_name) > 0
  )
  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    length(var.public_subnet_cidrs)
  )
}

resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${local.name_prefix}-igw"
  }
}

resource "aws_subnet" "public" {
  for_each = {
    for index, cidr in var.public_subnet_cidrs : tostring(index) => {
      cidr = cidr
      az   = local.availability_zones[index]
    }
  }

  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value.cidr
  availability_zone       = each.value.az
  map_public_ip_on_launch = true

  tags = {
    Name = "${local.name_prefix}-public-${each.key}"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = {
    Name = "${local.name_prefix}-public"
  }
}

resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "alb" {
  count = var.ecs_platform_enabled ? 1 : 0

  name        = "${local.name_prefix}-alb"
  description = "Allow public HTTP and HTTPS traffic to ALB."
  vpc_id      = aws_vpc.this.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "ecs_tasks" {
  count = var.ecs_platform_enabled ? 1 : 0

  name        = "${local.name_prefix}-ecs-tasks"
  description = "Allow ALB traffic to ECS tasks."
  vpc_id      = aws_vpc.this.id

  ingress {
    description     = "API from ALB"
    from_port       = var.container_port
    to_port         = var.container_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb[0].id]
  }

  ingress {
    description     = "AI from ALB"
    from_port       = var.ai_container_port
    to_port         = var.ai_container_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb[0].id]
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_s3_bucket" "alb_access_logs" {
  count = var.alb_access_logs_enabled ? 1 : 0

  bucket = "${local.name_prefix}-alb-access-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_public_access_block" "alb_access_logs" {
  count = var.alb_access_logs_enabled ? 1 : 0

  bucket = aws_s3_bucket.alb_access_logs[0].id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "alb_access_logs" {
  count = var.alb_access_logs_enabled ? 1 : 0

  bucket = aws_s3_bucket.alb_access_logs[0].id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "alb_access_logs" {
  count = var.alb_access_logs_enabled ? 1 : 0

  bucket = aws_s3_bucket.alb_access_logs[0].id

  rule {
    id     = "expire-alb-access-logs"
    status = "Enabled"

    filter {}

    expiration {
      days = var.alb_access_log_retention_days
    }
  }
}

resource "aws_s3_bucket_policy" "alb_access_logs" {
  count = var.alb_access_logs_enabled ? 1 : 0

  bucket = aws_s3_bucket.alb_access_logs[0].id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowAlbAccessLogDelivery"
        Effect = "Allow"
        Principal = {
          Service = "logdelivery.elasticloadbalancing.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.alb_access_logs[0].arn}/alb/AWSLogs/${data.aws_caller_identity.current.account_id}/*"
        Condition = {
          ArnLike = {
            "aws:SourceArn" = "arn:aws:elasticloadbalancing:${var.aws_region}:${data.aws_caller_identity.current.account_id}:loadbalancer/*"
          }
        }
      }
    ]
  })
}

resource "aws_s3_bucket" "waf_logs" {
  count = var.waf_count_enabled ? 1 : 0

  bucket = "aws-waf-logs-${local.name_prefix}-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_public_access_block" "waf_logs" {
  count = var.waf_count_enabled ? 1 : 0

  bucket = aws_s3_bucket.waf_logs[0].id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_server_side_encryption_configuration" "waf_logs" {
  count = var.waf_count_enabled ? 1 : 0

  bucket = aws_s3_bucket.waf_logs[0].id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "waf_logs" {
  count = var.waf_count_enabled ? 1 : 0

  bucket = aws_s3_bucket.waf_logs[0].id

  rule {
    id     = "expire-waf-logs"
    status = "Enabled"

    filter {}

    expiration {
      days = var.alb_access_log_retention_days
    }
  }
}

resource "aws_s3_bucket_policy" "waf_logs" {
  count = var.waf_count_enabled ? 1 : 0

  bucket = aws_s3_bucket.waf_logs[0].id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowWafLogDeliveryAclCheck"
        Effect = "Allow"
        Principal = {
          Service = "delivery.logs.amazonaws.com"
        }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.waf_logs[0].arn
        Condition = {
          StringEquals = {
            "aws:SourceAccount" = data.aws_caller_identity.current.account_id
          }
          ArnLike = {
            "aws:SourceArn" = "arn:aws:logs:${var.aws_region}:${data.aws_caller_identity.current.account_id}:*"
          }
        }
      },
      {
        Sid    = "AllowWafLogDeliveryWrite"
        Effect = "Allow"
        Principal = {
          Service = "delivery.logs.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.waf_logs[0].arn}/AWSLogs/${data.aws_caller_identity.current.account_id}/*"
        Condition = {
          StringEquals = {
            "aws:SourceAccount" = data.aws_caller_identity.current.account_id
            "s3:x-amz-acl"      = "bucket-owner-full-control"
          }
          ArnLike = {
            "aws:SourceArn" = "arn:aws:logs:${var.aws_region}:${data.aws_caller_identity.current.account_id}:*"
          }
        }
      }
    ]
  })
}

resource "aws_glue_catalog_database" "alb_access_logs" {
  count = var.alb_access_logs_enabled ? 1 : 0

  name = "${replace(local.name_prefix, "-", "_")}_alb_access_logs"
}

resource "aws_glue_catalog_table" "alb_access_logs" {
  count = var.alb_access_logs_enabled ? 1 : 0

  name          = "alb_access_logs"
  database_name = aws_glue_catalog_database.alb_access_logs[0].name
  table_type    = "EXTERNAL_TABLE"

  parameters = {
    EXTERNAL                       = "TRUE"
    "projection.enabled"           = "true"
    "projection.day.type"          = "date"
    "projection.day.range"         = "2026/07/01,NOW"
    "projection.day.format"        = "yyyy/MM/dd"
    "projection.day.interval"      = "1"
````
