# Troubleshooting

> 공개 이전: 2026-10-15. 원본은 `landit-iac.wiki`의 `3dd601a69d6cd66face489252793931ebf26cdb0`이다. 본문에 적힌 기준일과 검증 범위를 유지한 스냅샷이며, 현재 배포 상태를 새로 확인한 문서는 아니다. 내부 링크는 공개 근거로 바꾸거나 비공개 참조로 표시했다.


반복되는 production 인프라와 관측 문제를 증상별로 정리합니다. 장애가 진행 중이면 Incident Response Runbook (비공개 내부 문서)에서 영향, 담당과 승인 경계를 먼저 정합니다.

## 빠른 구분

| 증상 | 첫 확인 지점 | 관련 절차 |
| --- | --- | --- |
| Grafana 5xx 또는 telemetry missing | 원본 Grafana rule, health와 Overview | [Production 5xx와 telemetry missing](#production-5xx와-telemetry-missing) |
| Sentry 신규·회귀·급증 | Sentry event와 같은 시각의 BE·AI log | [Sentry 신규·회귀·급증 알림](#sentry-신규회귀급증-알림) |
| ECS 배포 정체 또는 task 반복 종료 | PRIMARY deployment, stopped reason | [ECS deployment 정체와 task 반복 종료](#ecs-deployment-정체와-task-반복-종료) |
| 개발 배포·환경변수 갱신 실패 | SSM 배포 문서, Compose, 실행 digest | [개발 EC2 배포 실패](#개발-ec2-배포-실패) |
| ALB target unhealthy | target health, health path와 startup log | [ALB target unhealthy](#alb-target-unhealthy) |
| SSM 변경 미반영 | parameter version, task definition mapping | [SSM 변경이 Container에 반영되지 않음](#ssm-변경이-container에-반영되지-않음) |
| Grafana metric 또는 log 없음 | app export, CloudWatch, Firehose와 Loki | [Grafana metric과 log 수집 공백](#grafana-metric과-log-수집-공백) |
| Discord 장애 알림 미수신 | 원본 rule 상태와 전달 경로 | [Discord 장애 알림 미수신](#discord-장애-알림-미수신) |
| 비정상 4xx 또는 WAF 차단 증가 | ALB access log와 WAF metric | [WAF와 ALB access log 조사](#waf-count와-alb-access-log-조사) |
| Push Queue 적체 또는 DLQ 발생 | Queue attribute, Consumer와 Alarm | [Push Queue 적체와 DLQ 발생](#push-queue-적체와-dlq-발생) |
| 체험 종료 이메일 미발송·미수신 | 실행 BE·DB 채널·상품 설정·SES | [체험 종료 이메일 미발송과 미수신](#체험-종료-이메일-미발송과-미수신) |
| Terraform plan 예상 밖 변경 | source, state, 변수와 provider | [Terraform plan에 예상하지 않은 변경이 있음](#terraform-plan에-예상하지-않은-변경이-있음) |
| backend initialization 또는 OIDC 오류 | 선택한 root, profile과 GitHub Environment | [Terraform backend와 OIDC 오류](#terraform-backend와-oidc-오류) |
| CloudFront 이미지 미반영 | object key, URL, DB 참조와 cache | [CloudFront 콘텐츠 미반영](#cloudfront-콘텐츠-미반영) |
| 관리자 이미지 업로드 CORS 오류 | S3 PUT CORS, origin과 서명 header | [관리자 이미지 업로드 실패](#관리자-이미지-업로드-실패) |

## Production 5xx와 telemetry missing

### 증상

- Grafana가 BE·AI `CRITICAL` 또는 `WARNING` 5xx를 알립니다.
- `MONITORING`이 runtime metric 부재를 알립니다.
- public health 또는 실제 사용자 기능이 실패합니다.

### 확인

1. 원본 rule의 service, severity, condition과 평가 시각을 확인합니다.
2. BE·AI health와 영향을 받은 사용자 경로를 분리해 확인합니다.
3. Overview에서 같은 시간대 5xx, latency, 배포 annotation과 로그를 봅니다.
4. Sentry event, ECS PRIMARY deployment와 ALB target health를 확인합니다.
5. `MONITORING`이면 애플리케이션 log가 계속 들어오는지와 OTLP export 오류를 함께 봅니다.
6. BE wrapper 오류라면 AI log, status와 ALB timeout을 같은 시각으로 비교합니다.

### 조치

- 애플리케이션, 배포, 설정, ALB와 telemetry 경계를 먼저 좁힙니다.
- health가 정상인 telemetry 공백이면 OTLP 설정과 metric export 경로를 확인합니다.
- 사용자 영향이 확인되면 Incident Response Runbook (비공개 내부 문서)의 승인 절차로 최소 완화를 선택합니다.
- 원인을 확인하지 않고 timeout을 늘리거나 같은 배포를 반복하지 않습니다.

### 복구 확인

- 사용자 경로와 BE·AI health가 정상입니다.
- ECS PRIMARY가 `COMPLETED`이고 ALB target이 `healthy`입니다.
- alert 평가 구간 동안 5xx와 latency가 정상으로 돌아옵니다.
- `MONITORING`이면 runtime metric과 log가 모두 다시 조회됩니다.

## Sentry 신규·회귀·급증 알림

### 증상

- `#alerts-sentry-prod`에 신규·회귀 issue가 도착합니다.
- 같은 issue가 짧은 시간에 반복돼 급증 알림이 도착합니다.

### 확인

1. 원본 Sentry event의 environment가 `prod`인지 확인합니다.
2. exception type, stack trace, cause, endpoint와 발생 시각을 확인합니다.
3. release 또는 배포 SHA를 실제 ECS task image와 비교합니다.
4. 같은 시각의 BE·AI log, Grafana 5xx·latency와 ALB 상태를 확인합니다.
5. wrapper 코드 아래에 lower-level exception과 외부 응답이 있는지 확인합니다.

### 조치

- 단일 event인지 지속 장애인지 발생 빈도와 사용자 영향을 확인합니다.
- 실제 예외가 애플리케이션 변경으로 재현되면 BE·AI 저장소의 hotfix와 release 절차를 따릅니다.
- 설정이나 인프라 문제라면 정확한 parameter 또는 resource를 정하고 변경 승인을 받습니다.
- issue 제목만 보고 model, provider 또는 bot traffic으로 분류하지 않습니다.

### 복구 확인

- 수정한 사용자 경로가 성공합니다.
- 같은 release와 환경에서 신규 event가 더 생기지 않습니다.
- Grafana와 CloudWatch의 같은 시간대 오류 신호가 정상으로 돌아옵니다.
- 임시 ignore, sample 또는 alert 억제를 사용했다면 제거 여부를 확인합니다.

## ECS deployment 정체와 task 반복 종료

### 증상

- PRIMARY deployment가 `IN_PROGRESS` 또는 `FAILED`에 머뭅니다.
- desired count보다 running count가 적습니다.
- 새 task가 시작 직후 반복 종료됩니다.

### 확인

1. service의 PRIMARY rollout, desired·running·pending과 최근 event를 확인합니다.
2. stopped task의 `stopCode`, `stoppedReason`과 container reason을 확인합니다.
3. CloudWatch startup log에서 첫 번째 애플리케이션 오류를 찾습니다.
4. task definition의 image, port, environment와 SSM mapping을 확인합니다.
5. ECR에 해당 image가 있고 ALB health path와 grace period가 맞는지 확인합니다.

### 조치

- `CannotPullContainerError`이면 ECR repository와 image tag를 맞춥니다.
- application start error이면 첫 stack trace의 원인을 애플리케이션 저장소에서 수정합니다.
- SSM 또는 권한 오류이면 task execution role, task role과 parameter mapping을 확인합니다.
- 같은 실패 배포를 반복 실행하기 전에 원인을 수정합니다.
- ECS 재배포, revision 교체와 scale 변경은 사용자 승인을 받은 뒤 실행합니다.

### 복구 확인

- PRIMARY deployment가 `COMPLETED`입니다.
- desired count와 running count가 일치합니다.
- 새 task가 안정화되고 이전 deployment가 정리됩니다.
- public health와 실제 사용자 경로가 성공합니다.

## 개발 EC2 배포 실패

### 증상

- 개발 API·AI 배포가 실패하거나 새 설정이 container에 반영되지 않습니다.
- AI container가 종료되거나 Docker 저장 공간이 부족합니다.

### 확인

1. 실행한 SSM 배포 문서의 버전·실패 단계와 필요한 SSM parameter 이름·타입을 확인합니다.
2. 실제 Compose image digest, API 768MiB·AI 1,024MiB 메모리 제한, 종료 원인과 CloudWatch log를 확인합니다.
3. EC2 디스크 사용량, 이전 이미지 기록과 cleanup timer를 확인합니다.
4. Caddy 외부 health와 API·AI loopback health를 구분합니다. BE의 AI 주소는 `http://ai:8000`입니다.

### 조치와 복구 확인

필수 parameter가 없으면 먼저 설정을 준비하고 최신 SSM 배포 문서로 서비스를 배포합니다. 임의의 `docker system prune`이나 volume 삭제를 하지 않습니다. 이미지 복구는 기록된 이전 digest를 사용하며, 설정 변경은 별도 복구합니다. 새 digest·env 반영·내외부 health와 실제 학습 요청까지 확인합니다. 정상 배포 절차 (비공개 내부 문서)를 따릅니다.

## ALB target unhealthy

### 증상

- ECS task는 running이지만 target group이 `unhealthy`입니다.
- public health가 5xx 또는 timeout을 반환합니다.

### 확인

1. target health의 reason과 description을 확인합니다.
2. API `/actuator/health`, AI `/health`, target port와 protocol을 확인합니다.
3. security group에서 ALB에서 task로 가는 port가 허용되는지 확인합니다.
4. container startup 완료 시각과 health check grace period를 비교합니다.
5. listener host rule이 올바른 target group으로 가는지 확인합니다.

### 조치

- path, port 또는 matcher가 코드와 다르면 Terraform plan으로 정확한 변경을 확인합니다.
- application이 기동하지 않았다면 ECS task log의 원인을 먼저 수정합니다.
- 기동 시간이 정상보다 길어졌다면 원인을 조사하고 실제 startup 시간 근거로 grace period를 검토합니다.
- listener, target group과 health 설정 변경은 saved plan과 승인을 거칩니다.

### 복구 확인

- 모든 active target이 `healthy`입니다.
- ECS PRIMARY deployment가 `COMPLETED`입니다.
- BE·AI public health가 `HTTP 200`을 반환합니다.
- 5xx와 target response time이 정상으로 돌아옵니다.

## SSM 변경이 Container에 반영되지 않음

### 증상

- parameter를 만들거나 값을 바꿨지만 애플리케이션 동작이 그대로입니다.
- task definition에 필요한 환경변수 이름이 없습니다.

### 확인

1. `/landit/develop` 또는 `/landit/prod`의 parameter name, type과 version을 확인합니다.
2. `docs/ssm-parameters.md` registry와 이름이 일치하는지 확인합니다.
3. 개발은 SSM 배포 문서와 runtime-env template, 운영은 실행 task definition의 `secrets` mapping을 확인합니다.
4. 개발은 갱신된 문서로 Compose container가 교체됐는지, 운영은 새 ECS deployment가 만들어졌는지 확인합니다.
5. CORS, AI 연결, Sentry 또는 해당 기능의 실제 사용 경로를 확인합니다.

### 조치

- parameter만 있고 mapping이 없으면 해당 환경의 runtime template 또는 task definition에 이름을 연결합니다.
- 기존 값만 바꾼 경우도 container 재배포가 필요합니다. mapping 변경이 없으면 값 변경 자체에 Terraform apply가 필요한 것은 아닙니다.
- develop과 production path가 뒤바뀌지 않았는지 확인합니다.
- actual value를 출력하거나 Wiki와 issue에 붙이지 않습니다.
- SSM 변경과 ECS 재배포는 정확한 대상과 영향 보고 뒤 승인받습니다.

### 복구 확인

- parameter name, type과 version이 예상과 같습니다.
- 개발 실행 env 또는 운영 실행 task definition에 같은 이름이 연결돼 있습니다.
- 개발 container 교체 또는 운영 PRIMARY deployment가 완료되고 기대한 image digest가 유지됩니다.
- 실제 endpoint 또는 smoke test에서 설정이 반영됩니다.

## Grafana metric과 log 수집 공백

### 증상

- Dashboard나 Explore에서 BE·AI metric이 보이지 않습니다.
- CloudWatch에는 log가 있지만 Loki에는 없습니다.
- `MONITORING` alert가 Firing합니다.

### 확인

1. 올바른 environment, service와 시간 범위를 선택했는지 확인합니다.
2. metric은 task의 OTLP endpoint·header와 애플리케이션 export log를 확인합니다.
3. log는 CloudWatch log group, subscription filter, Firehose delivery와 Loki label 순서로 확인합니다.
4. Firehose error와 application bucket의 `grafana-logs-failed/`를 확인합니다.
5. 최근 ECS deployment와 SSM rotation 시각을 비교합니다.

### 조치

- OTLP 설정이 빠졌다면 SSM mapping과 새 deployment를 준비합니다.
- CloudWatch에만 log가 있으면 subscription, Firehose와 Logs credential 경계를 확인합니다.
- metric은 없지만 health와 log가 정상이면 서비스 장애로 단정하지 않습니다.
- token과 secret ARN 전체를 출력하지 않습니다.
- Grafana, Secrets Manager, SSM과 ECS 변경은 승인 뒤 진행합니다.

### 복구 확인

- BE·AI metric series가 올바른 environment label로 조회됩니다.
- Loki에서 API·worker log group이 조회됩니다.
- Firehose delivery error와 새 failure backup이 없습니다.
- `MONITORING`이 Resolved된 뒤에도 health와 log를 따로 확인합니다.

## Discord 장애 알림 미수신

### 증상

- 원본 Grafana rule이나 Sentry alert는 발생했지만 Discord 메시지가 없습니다.
- Grafana Firing은 왔지만 Resolved가 오지 않습니다.

### 확인

1. 먼저 원본 Grafana rule 또는 Sentry issue rule이 실제로 조건을 만족했는지 확인합니다.
2. Grafana는 `alert_scope`, notification policy와 `discord-prod-incidents` contact point를 확인합니다.
3. Sentry는 Internal Integration action, API Gateway 응답, Lambda ingress와 delivery log를 순서대로 확인합니다.
4. relay payload의 environment가 명시적으로 `prod`인지 확인합니다.
5. rate limit, API allowlist, Lambda async retry와 Discord 응답 오류를 확인합니다.

### 조치

- Grafana route matcher, contact point와 template 연결을 계약 script로 검증합니다.
- Sentry relay는 signing secret과 webhook parameter 존재, Lambda IAM과 API Gateway policy를 값 없이 확인합니다.
- test alert 또는 test rule이 필요하면 정확한 대상과 삭제 계획을 정하고 승인받습니다.
- webhook URL과 signing secret을 출력하지 않습니다.

### 복구 확인

- Grafana test Firing과 Resolved가 올바른 채널에 도착합니다.
- Sentry test alert가 ingress와 delivery를 오류 없이 통과합니다.
- 임시 rule과 test alert가 정리됐습니다.
- 임시 service account와 token을 폐기했습니다.

## WAF Count와 ALB access log 조사

### 증상

- production 4xx가 증가하거나 특정 WAF rule Count가 증가합니다.
- 반복 요청의 경로와 분포를 확인해야 합니다.

### 확인

1. WAF Web ACL association과 각 rule의 실제 action을 확인합니다. Common Count 외에 label·rate Block과 IP reputation 그룹 action이 있습니다.
2. Common, IP reputation, common-label-block과 rate rule 중 어느 metric이 증가했는지 확인합니다.
3. sampled request와 같은 시간대 ALB access log를 비교합니다.
4. Athena에서 ALB status와 WAF action·terminating rule을 대조합니다. COUNT는 차단 결과가 아니며 BLOCK은 어느 rule이 종료했는지 확인합니다.
5. 정상 사용자, 회사·학교·통신사 공유 NAT 가능성을 확인합니다.

### 조치

- Count 증가는 차단 결과가 아니므로 사용자 영향과 실제 response status를 따로 확인합니다.
- client IP, user agent나 metric label만으로 bot 또는 특정 caller라고 단정하지 않습니다.
- query string과 사용자 정보가 포함된 결과를 공용 문서로 복사하지 않습니다.
- action·한도 변경은 예외 범위, 오탐 영향, rollback과 별도 승인이 있어야 합니다.

### 복구 확인

- 4xx와 Count 증가의 원인 경로가 확인됐습니다.
- 정상 사용자의 오탐과 공유 NAT 영향을 검토했습니다.
- 변경했다면 expected request만 영향을 받고 정상 사용자 경로가 성공합니다.
- WAF action, ALB association과 Terraform plan이 의도한 상태입니다.

### 외부 취약점 스캐너의 malformed request가 Sentry 500으로 보이는 경우

#### 증상

- 짧은 시간에 `MultipartException`의 `Stream ended unexpectedly` 또는 form URL decoding 예외가 Sentry 신규·회귀로 기록됩니다.
- 같은 시간대 ALB access log에 WordPress, PHP, 관리자 경로와 원격 실행 취약점 탐색이 대량으로 나타납니다.
- ALB 404와 460이 증가하지만 BE·AI health와 정상 사용자 경로는 유지됩니다.

#### 판단 기준

- multipart 본문이 종료 boundary 전에 끊기면 Tomcat이 `MultipartException`으로 변환할 수 있습니다.
- `application/x-www-form-urlencoded` 본문의 `%` escape가 깨지면 URL decoding 예외가 발생할 수 있습니다.
- ALB 460은 클라이언트가 idle timeout 전에 연결을 닫았다는 뜻이며, 잘린 요청 본문과 함께 나타날 수 있습니다.
- Sentry event에 요청 URL과 User-Agent가 없으면 event와 ALB 단일 요청을 1대1로 연결할 수 없습니다. 같은 task·시각·스캔 경로·예외 원인이 일치할 때만 외부 스캐너 상관관계를 높게 판단합니다.
- 이 신호는 Sentry 신규·회귀이지만, 자동으로 BE·AI 내부 장애 또는 사용자 영향으로 선언하지 않습니다.

#### 확인

1. Sentry event의 environment, 예외 체인, 발생 시각과 task identity를 확인합니다.
2. 같은 시각 ALB log의 404, 460, 5xx, 요청 경로와 User-Agent 분포를 비교합니다.
3. ECS target health, BE `/actuator/health`, AI `/health`와 실제 사용자 경로를 별도로 확인합니다.
4. WAF S3 log에서 `COUNT`·`BLOCK`, 관리형 rule label과 terminating rule을 확인합니다. `Authorization`, `Cookie`, `X-Api-Key`, query string의 원문은 조회 결과나 Wiki에 기록하지 않습니다.
5. Athena `waf_logs`에서 client IP, URI, action, terminating rule이 실제로 파싱되는지 행 수와 함께 검증합니다.

#### 조치

- `ip-rate-limit`은 IP당 5분 2,000회 `Block`을 유지합니다.
- Amazon IP Reputation List는 outer `override_action { none {} }`로 AWS 관리형 기본 action을 사용합니다. 이는 rule group 비활성화가 아니며, AWS가 Block으로 정의한 서브룰을 차단합니다.
- Common Rule Set은 전체 `Count`를 유지합니다. XSS body·query와 body size 계열은 정상 대화·학습 입력에 오탐될 수 있으므로, 서브룰별 근거 없이 group 전체를 Block으로 전환하지 않습니다.
- WAF rule 추가나 ALB timeout 변경으로 잘린 body를 직접 겨냥하지 않습니다. 경로 블랙리스트와 과도한 body 검사는 우회와 오탐 위험이 큽니다.
- BE는 `MultipartException`과 실제 request parsing 경로를 재현한 뒤, malformed multipart와 malformed form 요청을 400으로 명시 처리합니다. 모든 `IllegalArgumentException`을 일괄 400으로 바꿔 애플리케이션 버그를 숨기지 않습니다.

#### Athena WAF 로그가 조회되지 않는 경우

- `projection.log_time.format`이 `yyyy/MM/dd/HH/mm`이면 `projection.log_time.range` 시작값도 `yyyy/MM/dd/HH/mm` 형식이어야 합니다.
- 시작값이 `2026/07/25`처럼 일 단위이면 Athena가 파티션 값을 파싱하지 못합니다. `2026/07/25/00/00,NOW`처럼 분 단위로 맞춥니다.
- 변경은 Glue `waf_logs` table 한 건만 포함하는 Terraform saved plan으로 검토하고 승인 뒤 apply합니다.
- 수정 뒤 실제 WAF 로그 구간을 조회해 전체 행과 action, client IP, URI, terminating rule의 파싱 행 수가 같은지 확인합니다.

#### 롤백과 복구 확인

- IP Reputation 전환 뒤 정상 API의 403, 로그인·학습·대화 오류, 운영 서버의 AI 호출 오류를 관찰합니다.
- 정상 요청 차단이 확인되면 `aws-managed-ip-reputation`의 outer override를 `count {}`로 되돌리고, prod plan 검토와 승인 뒤 apply합니다.
- WAF 변경 뒤에는 public health, 실제 사용자 흐름, WAF S3 log와 post-apply Terraform plan을 함께 확인합니다.

## Push Queue 적체와 DLQ 발생

### 증상

- main Queue의 oldest message age 또는 visible count가 증가합니다.
- DLQ visible message가 생기거나 CloudWatch Alarm이 `ALARM`입니다.
- Push가 지연되거나 중복 발송됩니다.

### 확인

1. main Queue와 DLQ의 message count, not visible, retention, visibility와 redrive를 확인합니다.
2. Backend API Consumer가 실행 중이고 필요한 환경변수와 IAM 권한을 받았는지 확인합니다.
3. API log에서 메시지 유형, 비민감 오류 분류와 처리 시간을 확인합니다.
4. Scheduler state와 최근 실행 시각을 확인합니다.
5. 처리 시간이 300초 visibility timeout을 넘는지 확인합니다.
6. message body, Push token과 사용자 식별자는 조회하지 않습니다.

일회성 예약 전달 실패에는 Scheduler 오류 속성 `SCHEDULE_ARN`, `ERROR_CODE`가 있을 수 있습니다. 같은 DLQ의 Consumer 실패와 분리하고 예약 Target의 DLQ ARN·SendMessage 권한·캠페인 상태를 확인합니다. [단건 복구 기준](Push-Notifications.md#관리자-일회성-예약과-혼합-dlq)을 따릅니다.

### 조치

- Consumer 장애를 먼저 복구하고 backlog가 자연스럽게 감소하는지 확인합니다.
- 중복 전달과 순서 변경을 전제로 멱등 처리가 작동하는지 확인합니다.
- Scheduler가 원치 않게 활성화됐다면 대상과 영향 확인 뒤 비활성화 변경을 승인받습니다.
- Queue purge, 삭제와 재처리는 범위, 중복 발송과 복구 방법을 보고하고 승인받습니다. Scheduler 실패가 섞인 DLQ는 일괄 redrive하지 않습니다.

### 복구 확인

- main Queue oldest age와 visible count가 감소합니다.
- DLQ가 추가로 증가하지 않습니다.
- Consumer log에 반복 실패가 없습니다.
- 테스트 대상의 Push가 한 번만 발송되고 receipt check가 의도대로 처리됩니다.

## 체험 종료 이메일 미발송과 미수신

### 증상

- SES·IAM 적용 뒤에도 알림이 예약·발송되지 않습니다.
- BE에 ACCEPTED 이력이 있지만 메일함에서 확인되지 않습니다.

### 확인

1. 새 알림 BE 코드와 마이그레이션이 배포됐는지 확인합니다. 인프라 적용과 분리해 image SHA를 대조합니다.
2. DB 채널 ON/OFF, 실제 체험 상태·만료 시각, 연간 상품 ID와 sandbox 제외 설정을 확인합니다.
3. 실행 API의 SES 발신 주소·configuration set·상품 SSM 연결과 `notification-job-*` 예약 권한을 확인합니다.
4. SES identity·DKIM·production access, suppression과 전달 지표를 확인합니다. 수신 주소·사용자 원문은 공용 기록에 남기지 않습니다.

### 조치와 복구 확인

기존 알림을 중복 생성하지 말고 실패 단계의 설정·권한·코드를 수정합니다. SES 접수, 전달 지표와 실제 수신을 각각 확인합니다. [SES 운영 계약](Push-Notifications.md#ses와-체험-종료-알림)을 따르며 실제 메일 발송은 승인된 테스트 대상으로 제한합니다.

## Terraform plan에 예상하지 않은 변경이 있음

### 증상

- 요청 범위 밖 resource replacement, deletion, IAM 또는 network 변경이 보입니다.
- 같은 코드에서 plan drift가 반복됩니다.

### 확인

1. 선택한 root, backend와 state key를 확인합니다.
2. 최신 `origin/main`, 현재 branch와 로컬 diff를 확인합니다.
3. variable, provider version과 state drift를 확인합니다.
4. 전체 plan에서 resource 주소, action과 변경 속성을 읽습니다.
5. live AWS 정규화가 source와 다른지 read-only API로 비교합니다.
6. production은 실행 이미지 캡처 입력이 있는지, plan 후 deployment·revision·digest가 바뀌었는지 확인합니다.

### 조치

- 예상하지 않은 plan은 apply하지 않습니다.
- 이미지 보존 검사가 실패하면 현재 배포가 끝난 뒤 새 digest를 캡처하고 plan부터 다시 검토합니다. 검사를 우회하지 않습니다.
- source 오류, state drift, provider 정규화와 live 수동 변경 중 원인을 구분합니다.
- `-target`으로 다른 변경을 숨기지 않습니다.
- state 수정, import, resource 교체와 삭제는 별도 계획과 승인을 거칩니다.

### 복구 확인

- plan에는 요청한 resource와 연결된 action만 남습니다.
- 삭제와 replacement의 이유를 설명할 수 있습니다.
- apply했다면 후속 plan이 `No changes`입니다.
- 실제 resource와 사용자 경로가 정상입니다.

## Terraform backend와 OIDC 오류

### 증상

- `Backend initialization required`가 발생합니다.
- AWS CLI 인증 또는 GitHub Actions role assume이 실패합니다.
- workflow가 `Check AWS role variable`에서 중단됩니다.

### 확인

1. 선택한 root와 backend 설정을 확인합니다.
2. 로컬은 `landit` profile과 STS identity를 확인합니다.
3. GitHub Actions는 target별 `terraform-plan-*`, `terraform-apply-*` Environment를 확인합니다.
4. `AWS_ROLE_ARN` variable과 OIDC trust subject가 environment 이름과 맞는지 확인합니다.
5. production apply는 `main`, `confirm_environment=production`과 reviewer 승인을 확인합니다.

### 조치

로컬 backend를 다시 초기화합니다.

```bash
AWS_PROFILE=landit terraform -chdir=environments/dev init -reconfigure
```

root는 작업 환경에 맞게 바꿉니다. bootstrap 전 단계만 `init -backend=false`를 사용합니다.

OIDC 오류는 role ARN 존재, trust subject, repository·environment와 branch 조건을 순서대로 고칩니다. credential을 새로 만들거나 access key를 workflow에 넣지 않습니다.

### 복구 확인

- 선택한 root의 init과 validate가 성공합니다.
- STS identity가 예상한 Landit 계정입니다.
- plan job이 OIDC role을 assume하고 saved plan을 만듭니다.
- apply는 target environment 승인 뒤 private S3에서 받은 동일 saved plan의 SHA-256을 검증하고 실행합니다.

## 관리자 이미지 업로드 실패

### 증상

- 관리자 브라우저에서 OPTIONS가 거부되거나 presigned PUT이 실패합니다.

### 확인

S3 업로드 CORS의 origin·PUT·header 허용과 presigned URL의 Content-Type·Cache-Control·If-None-Match·만료를 확인합니다. `https://admin.landit.im`은 현재 source 기본 허용 목록에 있습니다. CloudFront GET CORS를 바꿔도 S3 PUT 정책은 바뀌지 않습니다.

### 조치와 복구 확인

실제 적용된 S3 CORS와 API의 `content/inbox/*` 권한을 확인하고 필요한 변경만 plan·승인 후 반영합니다. OPTIONS 성공, 실제 PUT, CloudFront 응답과 관리자 화면을 각각 확인합니다. 조회·업로드 정책 (비공개 내부 문서)을 따릅니다.

## CloudFront 콘텐츠 미반영

### 증상

- 새 이미지를 올렸지만 이전 이미지가 보입니다.
- CloudFront URL이 403·404를 반환하거나 API가 이전 URL을 응답합니다.

### 확인

1. object가 `content/*` 아래의 새 UUID key인지 확인합니다.
2. `Content-Type`과 immutable cache header를 확인합니다.
3. CloudFront URL의 status와 실제 object를 확인합니다.
4. develop·production DB의 참조 URL을 확인합니다.
5. browser cache와 최대 TTL을 확인합니다.
6. shared content, 사용자 콘텐츠와 Grafana failure backup을 구분합니다.

### 조치

- 같은 key를 덮어쓰지 않고 새 UUID key로 다시 게시합니다.
- OAC와 bucket policy가 해당 `content/*` object 읽기를 허용하는지 확인합니다.
- 새 URL 검증 뒤 DB 참조를 바꿉니다.
- 이전 object 삭제는 모든 참조 제거, TTL 경과와 사용자 승인 뒤 별도 작업으로 진행합니다.

### 복구 확인

- 새 CloudFront URL이 `HTTP 200`을 반환합니다.
- API와 frontend가 새 URL을 사용합니다.
- develop과 production에 이전 URL 참조가 없습니다.
- 삭제 대상이 다른 데이터 경로와 섞이지 않았습니다.
