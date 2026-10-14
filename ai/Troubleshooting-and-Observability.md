# Troubleshooting and Observability

> 공개 이전: 2026-10-15. 원본은 `landit-ai.wiki`의 `5700ea1bb57a7c6293f99d2a8398735cfb1488be`이다. 본문에 적힌 기준일과 검증 범위를 유지한 스냅샷이며, 현재 배포 상태를 새로 확인한 문서는 아니다. 내부 링크는 공개 근거로 바꾸거나 비공개 참조로 표시했다.


## 먼저 구분할 것

증상을 다음 범주 중 하나로 먼저 나눕니다.

| 증상 | 첫 확인 지점 |
| --- | --- |
| process가 시작되지 않음 | container log, Settings, OTEL 설정 |
| 400 | request DTO와 history 계약 |
| 401 | Backend header와 AI 내부 토큰 주입 여부 |
| 409 | message feedback cache와 호출 순서 |
| 502·503 | OpenRouter 응답 형식, provider·network·model 설정 |
| Free Talk 종료·표현 생성 실패 | `responseMode`, topic 계약, closing fallback log와 표현 참조 |
| 장기기억 생성·사용 누락 | WRITE·USE flag, 생성 상태, retrieval stage·fallback·trace |
| Pronunciation 실패 | Base64·FFmpeg, 기준 URL allowlist, 시간 예산과 정렬·판정 로그 |
| 202와 `feedbackStatus=FAILED` | message-feedback candidate와 repair 로그 |
| ECS 배포 정체 | PRIMARY deployment, task stop reason, service event |
| 품질 저하 | model·prompt·fixture 3회 결과 비교 |

HTTP status, 응답 body, 발생 시각, endpoint, 배포 SHA와 환경을 먼저 확보합니다. 사용자 원문과 secret은 ticket이나 공용 로그에 복사하지 않습니다.

## 서버 시작 실패

`OPENROUTER_API_KEY`와 `OPENROUTER_MODEL`은 app import나 `/health`가 아니라 실제 생성 호출 직전에 필요합니다. 서버가 시작도 못 한다면 다음 항목을 먼저 봅니다.

- Python 3.12와 dependency 설치가 정상인가.
- `OTEL_METRICS_ENABLED=true`인데 OTLP endpoint나 header가 비어 있지 않은가.
- `.env` 값의 문법과 boolean·float 변환이 유효한가.
- container command와 port가 Dockerfile·task definition과 맞는가.

`SENTRY_DSN`이 없으면 Sentry는 초기화되지 않지만 서버 시작 실패 원인은 아닙니다.

## OpenRouter 호출 실패

생성 API에서만 503이 난다면 다음 순서로 확인합니다.

1. API key와 기본 model이 container에 주입됐는지 확인합니다.
2. 메시지별 피드백만 실패하면 `MESSAGE_FEEDBACK_MODEL`과 review 설정을 확인합니다. Free Talk만 실패하면 `OPENROUTER_MODEL`과 임베딩 고정 모델 계약을, Pronunciation만 실패하면 `PRONUNCIATION_MODEL`과 시간 예산을 확인합니다.
3. OpenRouter provider·model 가용성과 rate limit, timeout을 확인합니다.
4. 같은 request를 비식별 fixture로 재현해 구조 실패인지 network 실패인지 구분합니다.
5. model을 바꾸기 전 최근 배포 SHA와 환경변수 변경 이력을 확인합니다.

API key, authorization header와 raw prompt 전문을 로그에 출력하지 않습니다.

## 400 요청 검증 실패

이 서버는 Pydantic `RequestValidationError`를 400 `INVALID_REQUEST`로 반환합니다. 먼저 응답 code를 확인하고 request를 `/openapi.json`과 해당 API 문서에 대조합니다. 특히 제출 메시지와 history의 마지막 USER 메시지, Conversation의 opening instruction, Free Talk의 topic·시간대·기억 ID 계약, Pronunciation의 문장·단어 토큰과 기준 URL allowlist, `expectedMessageIds` 중복을 확인합니다.

Pronunciation의 오디오 decode·길이·FFmpeg 실패는 400 `INVALID_AUDIO`입니다. DTO나 URL이 잘못된 `INVALID_REQUEST`와 구분합니다.

FastAPI가 자동 생성한 일반적인 422 설명만 보고 판단하지 말고 실제 공통 exception handler와 response body를 확인합니다.

## 401 내부 인증 실패

`LANDIT_AI_INTERNAL_TOKEN`이 설정된 AI는 `/api/` 요청의 `X-Landit-Internal-Token`을 본문 처리 전에 검사합니다. Backend의 header 전달 버전과 양쪽 runtime 주입을 확인합니다. 값은 출력하지 않고 일치 여부만 검증합니다. `/health` 성공만으로 내부 API 인증 성공을 판단하지 않습니다.

## 409 피드백 준비 실패

`MESSAGE_FEEDBACK_NOT_READY`는 `completedFeedbacks`가 없거나 `null`인 호환 경로에서 필요한 message ID가 cache에 없다는 뜻입니다. 완성 결과를 전달한 경로는 cache를 조회하지 않으며 세션·ID·순서 위반은 400입니다. HTTP 계약은 Conversation API (비공개 내부 문서), 저장·점수 규칙은 Feedback and Scoring (비공개 내부 문서)을 기준으로 확인합니다.

- 각 `message-feedback` 응답이 `PREPARING`이었는지 확인합니다.
- `FAILED`였던 message는 cache에 저장되지 않습니다.
- 요청의 `sessionId`와 `expectedMessageIds`가 실제 제출 결과와 같은지 확인합니다.
- TTL 만료, process 재시작, ECS task 교체와 다른 instance routing 여부를 확인합니다.
- Backend가 저장한 `completedFeedback`을 빠짐없이 순서대로 재전달했는지 확인합니다. 총평 성공은 cache 삭제 사유가 아닙니다.

## 502와 503

route에는 `AI_RESPONSE_INVALID` 502와 `AI_GENERATION_FAILED` 503 handler가 모두 있습니다. 현재 service의 실제 경계는 다음과 같습니다.

- JSON object를 파싱할 수 있고 알려진 field만 빠졌거나 잘못됐다면 endpoint별 fallback으로 복구할 수 있습니다.
- 모델 content가 비었거나 JSON object를 만들 수 없으면 recoverable wrapper가 생성 실패로 바꾸므로 보통 503입니다.
- provider SDK, network, 인증, model 설정 실패도 503입니다.
- 502는 복구 뒤에도 service가 `AiResponseInvalidError`를 route까지 올리는 경우를 위한 방어 경계입니다.

따라서 502를 단순히 OpenRouter 장애, 503을 단순히 형식 오류라고 고정해서 해석하지 않습니다. 응답 code와 같은 시각의 server log를 함께 봅니다.

Free Talk은 Conversation의 복구 wrapper를 사용하지 않습니다. 호출은 성공했지만 JSON object, 조건부 필드, 표현·기억 참조 또는 1536차원 임베딩 계약을 만족하지 않으면 보통 502입니다. provider·network·API key·모델 문제는 보통 503입니다.

`POST /api/v1/free-talk/closing`은 예외입니다. 모델 생성·계약 실패를 `free_talk_closing_fallback`으로 기록하고 검증된 종료 문구를 200으로 반환합니다. 종료 응답이 200이어도 같은 시각의 fallback log를 확인해야 모델 성공과 대체 응답을 구분할 수 있습니다.

Pronunciation은 모델 판정 형식 오류를 502로, 기준 음원 다운로드·정렬·필수 단계 시간 초과를 503으로 반환합니다. 기본 전체 예산 17초 안에 필수 판정과 강제 정렬이 끝나는지 확인하고, raw 음성이나 사용자 문장을 진단 로그에 추가하지 않습니다.

출력 형식 문제는 `structured_output_fallback`, `json_format_failure`, `schema_validation_failure`, `structured_output_retry`를 구분합니다. 수준 평가의 200과 `levelAssessment=null`은 유효한 Core를 얻지 못한 결과이며, `session_level_assessment_core_retry`·`session_level_assessment_core_failed`와 전체 시간 예산을 확인합니다.

## 장기기억 생성과 사용

장기기억 문제는 먼저 쓰기와 읽기를 나눕니다. 전체 상태·흐름은 [Long-Term Memory Architecture](Long-Term-Memory-Architecture.md)를 기준으로 확인합니다.

### 생성되지 않음

1. 배포된 Backend runtime에서 `LANDIT_MEMORY_WRITE_ENABLED`가 활성화됐는지 확인합니다. 값 자체를 ticket·로그에 복사하지 않습니다.
2. 완료된 Free Talk 세션의 `memory_generation_status`가 `NULL`, `PREPARING`, `READY`, `FAILED` 중 무엇인지 확인합니다.
3. `PREPARING`에 머물면 비동기 executor 등록·실행과 작업 시작 시각을 확인합니다.
4. `FAILED`면 `/memory-candidates`, 비교 검색, `/memory-resolution`, snapshot 재검증과 트랜잭션 중 어디서 실패했는지 확인합니다.
5. `READY`인데 기억이 없다면 후보가 0개였거나 모든 후보가 `IGNORE`였는지 구분합니다. 이는 정상 결과일 수 있습니다.

후보·판정·저장 실패는 부분 기억을 남기지 않는 fail-closed 경계입니다. 대화 완료 자체가 성공했다는 이유로 기억 저장까지 성공했다고 판단하지 않습니다.

### 생성 실패의 원인 좁히기

이 절은 2026-09-28 AI `c2ebd32`, BE `cfb15ec` 기준으로 보강했습니다. 전체 SHA와 재시도·시간 예산의 기준표는 [장기기억 실패 정책](Long-Term-Memory-Architecture.md#실패와-재시도-정책)에 있습니다.

1. 같은 세션·요청의 BE 실패 단계와 AI workflow를 연결합니다. 최종 HTTP 상태만으로 timeout인지 모델 판정 오류인지 단정하지 않습니다.
2. AI의 `memory_completion_attempts`에서 `first_failure`와 각 시도의 종료 사유·토큰 사용량·경과 시간을 확인합니다. 첫 오류 뒤 다른 오류가 발생해도 최초 원인을 보존해서 읽습니다.
3. 아래 분류에 따라 기존 복구가 수행됐는지 확인합니다. 곧바로 timeout이나 재시도 횟수를 늘리지 않습니다.

| 관측한 신호 | 확인할 것 |
| --- | --- |
| `completion_token_limit`, `finish_reason=length` | 추론·출력 토큰 소진 여부. 잘린 JSON을 정상 빈 후보로 처리하거나 동일 요청을 무조건 재전송하지 않습니다. |
| 후보 출처 오류 | 입력 USER ID 집합과의 검증 실패인지, 허용된 한 번의 교정도 실패했는지 확인합니다. 원문 전문을 로그에 추가하지 않습니다. |
| `duplicate_supersede` | 여러 후보가 같은 기억을 대체하려 했는지, 교정 결과도 계약 위반인지 구분합니다. |
| `format_unsupported` | provider의 명시적 형식 미지원에 따른 전환인지 확인합니다. 인증·network 오류와 구분합니다. |
| provider timeout·deadline·호출 한도 | 남은 공유 예산과 BE HTTP 제한을 함께 확인합니다. 개별 호출의 제한과 전체 비동기 작업 시간을 혼동하지 않습니다. |
| `STALE`·저장 실패 | 모델 생성 성공 이후에도 비교 snapshot 또는 저장 단계에서 거절될 수 있습니다. |
| `READY`와 저장 기억 0개 | 정상 빈 후보 또는 모든 후보 `IGNORE`인지 확인합니다. 실패를 숨긴 빈 결과라고 단정하지 않습니다. |

실패 상태 기록 자체의 오류는 `failure_state_persistence / storage_failed`로 별도 관측합니다. 생성 작업이 실패했다고 DB 상태가 반드시 `FAILED`로 저장됐다고 가정하지 않습니다. 후속 질문만 fallback이 된 경우도 필수 기억 생성 실패와 구분합니다.

### 다음 대화에서 사용되지 않음

1. 배포된 Backend runtime에서 `LANDIT_MEMORY_USE_ENABLED`가 활성화됐는지 확인합니다.
2. 세션 시작 방식에 따라 retrieval stage가 `OPENING` 또는 `FIRST_USER_TURN`인지 확인합니다.
3. `free_talk_memory_retrieval`의 rank 0 marker로 해당 stage가 이미 선점됐는지 확인합니다.
4. `landit.memory.fallback` counter와 같은 stage의 query embedding·pgvector·trace 오류를 확인합니다.
5. 후보 row의 rank·distance가 있는데 `used=false`라면 검색은 됐지만 AI가 응답 생성에 사용하지 않은 경우입니다.
6. 현재 검색 정책은 `memory-retrieval-v2`, 거리 상한은 0.8, query embedding 요청 timeout은 2초입니다. 후보 없음과 timeout fallback을 구분합니다.
7. `used=true`인데 사용자 경험이 이상하다면 `response_message_id`를 기준으로 승인된 비식별 재현 자료를 만들고 품질 문제로 조사합니다.

검색 실패는 빈 `memoryContext`로 대화를 계속하는 fail-open 경계입니다. 정상 200과 기억 없는 응답만으로 flag 비활성, 후보 없음, 검색 실패, 미사용 중 어느 경우인지 구분할 수 없습니다.

과거 기억을 현재형으로 답한다면 요청 timezone, `validFrom`·`validTo`·`observedAt`, 서버 현재 시각과 계산된 시간 상태를 확인합니다. `usedMemoryIds`는 사용 여부이며 시간 해석의 정확성을 보장하지 않습니다.

## feedbackStatus FAILED

`POST /message-feedback`의 202는 HTTP 요청 접수 결과이며, body의 `feedbackStatus=FAILED`는 피드백 생성 실패를 뜻합니다. 생성·복구·검수의 상세 조건은 Feedback and Scoring (비공개 내부 문서)을 기준으로 확인합니다.

log의 `workflow=message_feedback_failed`와 `reason`을 확인합니다. `FAILED`를 받은 backend는 해당 message를 완료된 피드백으로 간주하거나 곧바로 session feedback을 호출하면 안 됩니다.

응답이 `PREPARING`인데 `workflow=message_feedback_copy_fallback`이 있다면 검수 대신 생성 단계 candidate를 사용한 경우입니다. 같은 fixture에서 반복되는지 품질 평가 결과와 함께 확인합니다.

## Cache와 다중 인스턴스

cache의 구조와 TTL은 System Architecture (비공개 내부 문서)를 기준으로 확인합니다. 운영 중 다음 사건이 409와 같은 시각에 있었는지 조사합니다.

- ECS task 재시작·교체.
- process 종료.
- 3시간 TTL 만료.
- 다른 instance로 요청 routing.

현재는 Backend가 `completedFeedbacks`를 재전달하는 경로가 구현되어 있습니다. cache 문제가 있다면 먼저 이 payload가 실제 전송되는지 확인합니다. 임시로 TTL만 늘려 instance 불일치를 숨기지 않습니다.

## Sentry

`SENTRY_DSN`이 있을 때만 Sentry SDK를 초기화합니다. `APP_ENV`가 environment로 들어가고 trace sample rate 기본값은 0입니다.

Sentry에서 다음 정보를 연결합니다.

- 발생 시각과 environment.
- exception type과 stack trace.
- release·배포 SHA를 확인할 수 있는 배포 기록.
- 같은 시간대 ECS task와 service event.

raw prompt, API key, authorization header와 사용자 원문을 custom context로 추가하지 않습니다.

Pronunciation 요청의 `userAudio`는 Sentry event에서 제거됩니다. 오디오 payload 누락·decode 문제를 조사할 때도 raw Base64를 attachment나 breadcrumb로 남기지 않습니다.

## OpenTelemetry

`OTEL_METRICS_ENABLED=false`가 기본입니다. 활성화하면 OTLP endpoint와 header가 모두 필요합니다.

현재 수집 범위는 다음과 같습니다.

- `/health`를 제외한 HTTP route template, method, status, error type.
- process CPU·memory·thread.
- CPython GC.

request·response body, header, query string과 사용자·세션·메시지 ID는 metric attribute로 수집하지 않습니다. route 미매칭 요청에는 `http.route` label을 붙이지 않습니다. metric만으로 모델 판정 이유나 특정 사용자 요청 원문을 알 수는 없습니다.

## ECS 배포 정체

workflow가 안정화를 기다리다 실패하면 다음 순서로 확인합니다.

1. ECS service의 PRIMARY `rolloutState`, desired·running·pending count와 deployment 수.
2. 최근 service event.
3. stopped task의 `stopCode`, `stoppedReason`과 container reason.
4. CloudWatch startup log.
5. task definition의 image SHA, environment·secret 주입과 health check.
6. ECR에 해당 SHA image가 실제로 존재하는지.

`deployment-record.json`의 배포 전후 revision·digest와 새 deployment ID를 실제 task와 대조합니다. service가 안정화되어도 대상 revision이나 digest가 다르거나 health 호출이 실패하면 workflow는 실패합니다.

PRIMARY가 `FAILED`이거나 새 task가 계속 종료되면 같은 workflow를 반복 실행하기 전에 원인을 수정합니다.

## Develop EC2 배포 실패

`Deploy dev worker`는 ECS 안정화가 아니라 SSM Run Command 결과를 확인합니다.

1. workflow가 선택한 ref와 `${GITHUB_SHA}`를 확인합니다.
2. ECR에 해당 SHA image가 존재하는지 확인합니다.
3. SSM command ID의 status가 `Success`, `Failed`, `TimedOut`, `Cancelled` 중 무엇인지 확인합니다.
4. 대상 EC2 instance가 online이고 `develop-landit-ec2-deploy` document를 실행할 권한이 있는지 확인합니다.
5. EC2의 AI container가 같은 SHA image와 최신 runtime env로 재기동됐는지 확인합니다.

SSM command 성공은 `/health`나 대표 API E2E 성공을 뜻하지 않습니다. 배포 후 별도로 확인합니다.

## 로그에 남기면 안 되는 정보

- OpenRouter·AWS·Sentry·OTLP credential.
- authorization header와 cookie.
- raw production prompt와 모델 response 전문.
- 사용자 원문, 번역문과 개인 식별 정보.
- `.env`, SSM secret value와 private key.

진단 로그는 `workflow`, 실패 `reason`, endpoint, provider, model, request correlation에 필요한 최소 식별자처럼 비민감 메타데이터를 중심으로 남깁니다. 실제 데이터를 재현해야 한다면 승인된 비식별 fixture를 사용합니다.
