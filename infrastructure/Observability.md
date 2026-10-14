# Observability and Alerting

> 공개 이전: 2026-10-15. 원본은 `landit-iac.wiki`의 `3dd601a69d6cd66face489252793931ebf26cdb0`이다. 본문에 적힌 기준일과 검증 범위를 유지한 스냅샷이며, 현재 배포 상태를 새로 확인한 문서는 아니다. 내부 링크는 공개 근거로 바꾸거나 비공개 참조로 표시했다.


Landit은 Sentry에서 exception issue를 확인하고 Grafana Cloud에서 애플리케이션 지표와 로그를 함께 봅니다. Production 장애 알림은 Sentry와 Grafana가 서로 다른 Discord 경로로 전달합니다.

## 먼저 볼 곳

| 신호 | 첫 확인 지점 | 다음 확인 |
| --- | --- | --- |
| Grafana `CRITICAL` 또는 `WARNING` | 알림의 Grafana rule과 Overview dashboard | 같은 시각 BE·AI 로그, Sentry, ECS와 ALB |
| Grafana `MONITORING` | runtime metric과 Loki log 수집 여부 | 서비스 장애와 telemetry 경로를 분리 |
| Sentry 신규·회귀 issue | Sentry event, stack trace, environment와 release | 같은 시각 Grafana, CloudWatch와 배포 SHA |
| Discord 알림이 오지 않음 | 원본 Sentry·Grafana rule 상태 | notification route, relay 또는 contact point |
| 4xx 분포와 WAF 차단 조사 | production ALB access log와 WAF metric | Athena WAF log의 action·terminating rule |
| Queue 적체와 DLQ | AWS SQS·CloudWatch attribute | [Push Notifications](Push-Notifications.md) 또는 jobs Queue 담당 |
| 이메일 접수 뒤 미수신 | SES identity·suppression·configuration set 지표 | BE 발송 이력과 실제 수신 결과 |

장애 대응을 시작했다면 Incident Response Runbook (비공개 내부 문서)에 발생 시각, 환경, 서비스, 사용자 영향과 원본 링크부터 기록합니다.

## Production 장애 알림

| 경로 | Discord | 보내는 신호 |
| --- | --- | --- |
| Grafana Alerting | `#alerts-grafana-prod` | BE·AI 5xx와 telemetry missing의 Firing·Resolved |
| Sentry issue alert | `#alerts-sentry-prod` | production BE·AI의 신규·회귀 issue와 반복 급증 |

Discord 메시지는 원인을 확정하지 않습니다. 알림의 상세 링크에서 원본 rule이나 event를 열고 같은 시간대의 다른 증거를 연결합니다.

## Grafana Alerting

### Production 규칙

| 서비스 | 심각도 | 조건 | Pending |
| --- | --- | --- | --- |
| BE | `CRITICAL` | 최근 2분 5xx가 3건 이상이고 오류율이 50% 이상 | 1분 |
| AI | `CRITICAL` | 최근 2분 5xx가 3건 이상이고 오류율이 50% 이상 | 1분 |
| BE | `WARNING` | 최근 10분 5xx가 10건 이상이고 오류율이 20% 이상 | 3분 |
| AI | `WARNING` | 최근 10분 5xx가 10건 이상이고 오류율이 20% 이상 | 3분 |
| BE | `MONITORING` | `jvm_threads_live`가 10분 동안 없음 | 5분 |
| AI | `MONITORING` | `process_thread_count`가 10분 동안 없음 | 5분 |

모든 규칙은 1분마다 평가합니다. BE `/actuator`와 AI `/health`는 5xx 비율의 분자와 분모에서 제외합니다.

`MONITORING`은 서비스 장애를 뜻할 수도 있고 OTLP 수집 공백일 수도 있습니다. health, 애플리케이션 로그, 최근 배포와 metric export 상태를 함께 확인합니다.

### Notification policy

규칙에는 `environment=prod`, `service`, `severity`, `alert_scope=landit_incident` label이 있습니다. policy는 `alert_scope=landit_incident`만 `discord-prod-incidents` contact point로 보내고 `service`, `severity`로 묶습니다.

- 첫 알림은 30초 뒤 전송합니다.
- 같은 그룹의 추가 알림은 5분 간격으로 묶습니다.
- 계속 Firing이면 1시간마다 다시 전송합니다.
- Firing과 Resolved를 모두 전송합니다.

Resolved 알림을 받아도 Incident Response Runbook (비공개 내부 문서)의 복구 조건을 따로 확인합니다.

## Sentry Alerting

Sentry는 production BE·AI project의 신규·회귀 issue와 반복 급증을 알립니다. 신규·회귀는 조사 시작 신호이며, 같은 issue가 5분 동안 10회 이상 발생하면 급증 조건을 확인합니다. 같은 rule의 반복 알림은 issue별 30분 간격으로 제한합니다. Sentry resolved 상태는 Discord로 보내지 않습니다.

전달 경로는 다음과 같습니다.

```text
Sentry Internal Integration alert action
  -> API Gateway async Lambda integration
  -> prod Sentry Discord relay Lambda ingress
  -> 같은 Lambda의 비동기 delivery invocation
  -> #alerts-sentry-prod
```

API Gateway는 Sentry outbound IP만 허용하고 초당 1건, burst 5건으로 제한합니다. Lambda ingress는 서명 형식과 body 크기를 확인한 뒤 delivery를 비동기로 호출합니다. Delivery가 SSM의 signing secret으로 HMAC-SHA256을 검증하고 environment가 명시적으로 `prod`인 payload만 전달합니다.

Lambda는 최대 event age 5분과 재시도 2회를 사용합니다. 실패 destination과 DLQ는 현재 없으므로 재시도를 모두 소진한 alert는 재처리할 수 없습니다. Discord 미수신 시 Sentry rule, API Gateway, Lambda ingress와 delivery log를 순서대로 확인합니다.

Sentry issue 제목이나 `AI_GENERATION_FAILED` 같은 wrapper 코드만으로 model, provider, BE 또는 AI 실패라고 단정하지 않습니다. stack trace, cause, 같은 시각의 BE·AI 로그, ALB 응답과 timeout 경계를 비교합니다.

## 지표와 Dashboard

`Landit` folder에는 세 dashboard가 있습니다.

- [Landit Overview](https://scarletmyrtle3008.grafana.net/d/landit-overview/landit-overview). BE·AI의 TPS, 5xx, 응답시간, 7일 전 비교와 통합 로그를 확인합니다.
- [Landit BE](https://scarletmyrtle3008.grafana.net/d/landit-be/landit-be). endpoint, JVM, Tomcat, HikariCP와 API 로그를 확인합니다.
- [Landit AI](https://scarletmyrtle3008.grafana.net/d/landit-ai/landit-ai). endpoint, process, CPython GC와 AI 로그를 확인합니다.

dashboard는 `prod`와 `develop`을 전환하며 시간대는 `Asia/Seoul`입니다. `workflow=deployment_started` 로그의 `serviceVersion`을 배포 annotation에 사용합니다. incident 시각과 최근 배포가 겹치면 annotation의 Git commit SHA를 BE·AI 저장소에서 확인합니다.

BE와 AI는 OTLP로 지표만 직접 전송합니다. 공통 resource attribute는 namespace, environment, service name과 Git commit SHA입니다. SQL, 사용자 식별자, request·response body와 header는 metric label로 보내지 않습니다.

## 로그와 ALB 요청 분석

애플리케이션 로그 경로는 다음과 같습니다.

```text
BE·AI stdout
  -> CloudWatch Logs
  -> Amazon Data Firehose
  -> Grafana Loki
```

Loki에서는 `project`, `environment`, `aws_log_group` label을 사용합니다. BE 에러는 Spring console의 `ERROR`, `FATAL` 위치를 확인합니다. AI 에러는 logfmt `level`의 `ERROR`, `CRITICAL`만 사용합니다. message에 `error`라는 단어가 있다는 이유로 오류 로그로 분류하지 않습니다.

production ALB access log는 private S3 bucket의 `alb/`에 저장하고 30일 뒤 만료합니다. Glue table은 일별 partition projection을 사용합니다. Athena named query는 최근 3일 4xx를 KST 시각, client IP, method, URL, ALB·target status와 user agent로 최대 1,000건 조회합니다.

ALB URL에는 query string이 포함될 수 있습니다. 분석 결과를 metric label이나 공용 문서로 복사하지 않습니다. access log의 client IP와 user agent는 요청 근거이며 실제 사람이나 bot 신원을 확정하는 증거가 아닙니다.

## WAF 규칙과 로그

Production ALB에는 기본 허용 REGIONAL Web ACL이 연결돼 있습니다.

| 규칙 | origin/main의 action |
| --- | --- |
| `aws-managed-common` | Common Rule Set 결과를 Count로 override |
| `aws-managed-ip-reputation` | `override_action=none`, 관리형 그룹의 원래 action 유지 |
| `common-label-block` | `RestrictedExtensions_URIPath`, `BadBots_Header`, `GenericLFI_URIPath` label을 Block |
| `ip-rate-limit` | 동일 IP 300초 구간, 기본 한도 2,000회 초과 시 Block |

리소스 이름의 `alb-count`와 변수명 `waf_count_enabled`는 모든 규칙이 Count라는 의미가 아닙니다. 이 표는 코드 구성입니다. 현재 AWS action과 트래픽 차단 결과는 Web ACL과 같은 시각의 WAF log에서 따로 확인합니다.

WAF log는 전용 private S3에 저장하고 COUNT·BLOCK만 KEEP, 나머지는 DROP합니다. Authorization·Cookie·X-Api-Key와 query string을 redaction하며 Glue·Athena로 action과 terminating rule을 조사합니다. redaction은 sampled request 전체의 비식별화를 보장하는 설명이 아니므로 원문을 공용 기록에 복사하지 않습니다.

규칙·한도를 바꿀 때는 ALB access log, WAF metric과 공유 NAT 영향을 비교하고 요청 범위와 승인을 확인합니다. [Terraform 규칙과 logging 설정](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-iac/6e85e5f70b61919fb89fc777c10699eb9549a9a5/modules/app-platform/main.tf.md)을 기준으로 합니다.

## SES 전달 지표

환경별 transactional configuration set은 `AWS/SES`의 `Environment` dimension으로 SEND·DELIVERY·BOUNCE·COMPLAINT·REJECT·DELIVERY_DELAY를 기록하고 BOUNCE·COMPLAINT suppression을 사용합니다. 이 구성에는 개별 이벤트 소비자나 경보 수신처가 없습니다.

BE의 `ACCEPTED`, SES 접수, DELIVERY 집계와 실제 메일함 수신은 서로 다른 증거입니다. 체험 알림의 BE 배포·DB 채널·대상 상품과 sandbox 설정도 함께 확인합니다. [SES와 체험 종료 알림](Push-Notifications.md#ses와-체험-종료-알림)의 기준을 따릅니다.

## 인증과 비밀정보

- OTLP 인증 header는 환경별 SSM `SecureString`에서 BE·AI task에 주입합니다.
- Firehose Logs 인증값은 Terraform 밖의 Secrets Manager에 보관합니다.
- Sentry signing secret과 Discord webhook URL은 production SSM `SecureString`에 보관합니다.
- Dashboard와 alert 설정 변경에는 유효 기간을 제한한 Grafana service account token을 환경변수로만 전달합니다.
- 검증이 끝나면 임시 service account와 token을 폐기합니다.

token, Authorization header, webhook URL, secret ARN, raw production prompt와 사용자 원문을 Wiki, issue와 공용 로그에 기록하지 않습니다.

## 변경 후 검증

저장소의 계약 검사를 먼저 실행합니다.

```bash
./scripts/test-grafana-prod-alert-contract.sh <rule-group.json> <notification-policy.json>
./scripts/test-grafana-discord-template-contract.sh <notification-template.json> <contact-point.json>
./scripts/test-athena-alb-contract.sh
./scripts/test-grafana-analysis-panels.sh
```

live 변경을 승인받아 적용했다면 다음 항목을 확인합니다.

1. BE·AI service가 새 설정으로 안정화됐습니다.
2. Grafana query와 alert rule이 Error 상태가 아닙니다.
3. test rule의 Firing과 Resolved가 올바른 Discord 채널에 도착합니다.
4. Sentry test alert가 API Gateway와 Lambda delivery를 거쳐 도착합니다.
5. ALB access log object와 WAF Count metric이 생성됩니다.
6. 임시 rule, service account와 token을 제거했습니다.

## 현재 제약

- 조직 정책이 `tag:GetResources`를 막아 Grafana CloudWatch integration을 사용하지 않습니다.
- ALB TPS, ECS CPU·memory와 SQS 적체·DLQ 수는 현재 Grafana Cloud 수집 범위 밖입니다.
- Sentry relay에는 실패 destination과 DLQ가 없습니다.
- Grafana와 Sentry의 live rule 상태는 인증된 조회가 필요합니다. source 계약만 보고 현재 Firing 여부를 단정하지 않습니다.
- metric과 log label은 원본 사용자 요청이나 caller 신원을 제공하지 않습니다.
