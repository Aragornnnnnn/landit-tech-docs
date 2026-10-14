# email.tf — 공개 근거 발췌

- 원본 저장소: `landit-iac` (비공개).
- 원본 커밋: `6e85e5f70b61919fb89fc777c10699eb9549a9a5`.
- 원본 경로: `environments/dev/email.tf`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–81행


````hcl
# 개발 서버의 SES 발신 도메인과 메일 이벤트 지표, 최소 발송 권한을 정의한다.
resource "aws_sesv2_email_identity" "landit" {
  email_identity = "landit.im"

  dkim_signing_attributes {
    next_signing_key_length = "RSA_2048_BIT"
  }
}

resource "aws_sesv2_configuration_set" "transactional" {
  configuration_set_name = "${local.name_prefix}-transactional"

  reputation_options {
    reputation_metrics_enabled = true
  }

  suppression_options {
    suppressed_reasons = ["BOUNCE", "COMPLAINT"]
  }
}

resource "aws_sesv2_configuration_set_event_destination" "metrics" {
  configuration_set_name = aws_sesv2_configuration_set.transactional.configuration_set_name
  event_destination_name = "delivery-metrics"

  event_destination {
    enabled              = true
    matching_event_types = ["SEND", "DELIVERY", "BOUNCE", "COMPLAINT", "REJECT", "DELIVERY_DELAY"]

    cloud_watch_destination {
      dimension_configuration {
        default_dimension_value = var.environment
        dimension_name          = "Environment"
        dimension_value_source  = "MESSAGE_TAG"
      }
    }
  }
}

data "aws_iam_policy_document" "ec2_email" {
  statement {
    actions = ["ses:SendEmail"]
    resources = [
      aws_sesv2_email_identity.landit.arn,
      aws_sesv2_configuration_set.transactional.arn
    ]
    condition {
      test     = "StringEquals"
      variable = "ses:FromAddress"
      values   = ["no-reply@landit.im"]
    }
  }

  statement {
    actions = ["scheduler:CreateSchedule", "scheduler:GetSchedule"]
    resources = [
      "arn:aws:scheduler:${var.aws_region}:${data.aws_caller_identity.current.account_id}:schedule/${module.app_platform.admin_push_scheduler_group}/notification-job-*"
    ]
  }
  # 기존 admin-push-manage 정책이 같은 Scheduler 역할의 iam:PassRole을 제공한다.
}

resource "aws_iam_role_policy" "ec2_email" {
  name   = "${local.name_prefix}-email"
  role   = aws_iam_role.ec2_app.id
  policy = data.aws_iam_policy_document.ec2_email.json
}

output "ses_dkim_records" {
  description = "Vercel DNS에 추가할 SES 도메인 인증 CNAME. 발신 도메인은 이 root에서 한 번만 소유한다."
  value = [for token in aws_sesv2_email_identity.landit.dkim_signing_attributes[0].tokens : {
    name  = "${token}._domainkey.landit.im"
    type  = "CNAME"
    value = "${token}.dkim.amazonses.com"
  }]
}

output "ses_configuration_set" {
  description = "개발 서버의 이메일 전달, 반송 및 불만 CloudWatch 지표 설정."
  value       = aws_sesv2_configuration_set.transactional.configuration_set_name
}
````
