# 푸시 발송·중복 방지·재시도 정책

> 공개 이전: 2026-10-15. 원본은 `landit-be.wiki`의 `2ee6703b97886858f72bc6ba7c6c7ffe381689ae`이다. 본문에 적힌 기준일과 검증 범위를 유지한 스냅샷이며, 현재 배포 상태를 새로 확인한 문서는 아니다. 내부 링크는 공개 근거로 바꾸거나 비공개 참조로 표시했다.


이 문서는 정기 학습 알림과 일반 푸시의 발송 정책을 설명한다. Queue·Scheduler 설정은 [IaC Push Notifications](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/infrastructure/Push-Notifications.md)가 담당한다. 관리자 공지는 실패 처리 방식이 달라 아래에 별도로 구분한다.

| 기준 | 값 |
| --- | --- |
| 확인 날짜 | 2026-09-28 |
| BE 코드 | `origin/main`의 `cfb15ec066277ed0baa255395c02c6c0500e99d7` (비공개 원본 참조) |
| 확인 범위 | 발송 코드·기존 테스트 대조. 이번 문서 갱신에서 테스트 실행, 운영 발송, 기기 수신은 확인하지 않음. |

과거 블로그의 키 구성이나 수치는 당시 구현·실험의 기록이다. 현재 정책은 이 문서의 기준 커밋을 따른다. 코드가 지원한다는 사실과 실제 Scheduler가 활성화돼 있다는 사실은 별개다.

## 발송 흐름과 처리 단위

`예약 메시지 수신 → 대상 사용자 조회 → 발송 이력 선점 → Expo 제출 → Ticket 저장 → Receipt 확인 예약` 순서로 처리한다. **선점**은 외부 요청 전에 DB에서 이번 발송을 처리할 권한을 확보하는 단계다.

| 작업 | 처리 단위 | 의미 |
| --- | --- | --- |
| 정기 알림 대상 조회 | 사용자 500명씩 keyset 조회 | 전체 대상을 한 번에 메모리에 올리지 않는다. |
| 발송 준비·Ticket 저장 | 최대 100건씩 별도 트랜잭션 | 토큰·이력 잠금과 상태 검증을 유지하며 저장을 묶는다. |
| Expo 제출 | 최대 100건 | DB에서 실제 선점한 발송만 모아 보낸다. |
| Receipt 예약 | SQS 요청당 최대 10개 메시지 | HTTP 성공뿐 아니라 항목별 성공 ID와 실패 목록도 검사한다. |

DB 준비 후보 100건과 실제 발송 100건은 다르다. 이미 처리한 후보는 제외되므로, 준비 결과를 다음 묶음과 모아서 Expo에 제출한다. 이 크기들은 요청당 처리 단위이며 초당 전송률이나 최적 성능을 보장하는 값이 아니다.

## 중복 방지 키

| 구분 | 키 |
| --- | --- |
| 정기 알림 이벤트 | `scheduled:{예정 시각의 KST 날짜}:{userProfileId}` |
| 일반 발송 이력 | `push:{eventId}:{userPushTokenId}` |
| 정기 알림의 최종 키 | `push:scheduled:{KST 날짜}:{userProfileId}:{userPushTokenId}` |

정기 알림 키에는 **알림 유형이 포함되지 않는다.** 같은 날짜의 같은 사용자·토큰에 대한 재처리에서 알림 유형이 달라져도 다른 발송으로 취급하지 않는다. Scheduler 실행 시도 ID도 최종 발송 키로 사용하지 않는다.

토큰은 Expo 문자열이 아니라 DB의 `userPushTokenId`로 구분한다. 따라서 이 정책은 사용자에게 평생 한 번, 또는 사용자당 하루 한 기기에만 보낸다는 뜻이 아니다. 활성 토큰이 여러 개면 각각 발송 대상이 될 수 있다.

준비 단계는 기존 이력 잠금, 발송 가능한 토큰의 소유자·상태 재확인, 토큰 잠금 후 키 재조회를 수행한다. 신규 키에는 DB 고유 제약도 적용한다. 이력이 이미 있으면 명시적으로 재시도 가능한 경우만 다시 선점한다.

## 발송 상태

| 상태 | 의미 | 자동 재발송 |
| --- | --- | --- |
| `REQUESTED` | 발송 이력을 확보함. 외부 접수 성공을 뜻하지 않음. | 재시도 표식이 있고 기존 Expo 토큰도 일치할 때만 다시 선점. |
| `TICKET_ACCEPTED` | Expo가 요청을 접수했고 Ticket ID가 저장됨. | 재발송하지 않고 Receipt 확인을 복구. |
| `DELIVERED` | Expo Receipt가 성공을 반환함. | 재발송하지 않음. |
| `FAILED` | Ticket·Receipt 오류 또는 확인 불가능 상태를 종료 처리함. | 일반 발송 경로에서 자동 재발송하지 않음. |

재시도는 별도 상태가 아니라 `REQUESTED`의 오류 코드 표식으로 관리한다. 잠금 안에서 표식을 지워 한 처리 흐름만 재시도하도록 한다. `DELIVERED`는 기기 화면 표시나 사용자의 알림 열람을 증명하지 않는다.

## 일반 발송의 실패 처리

| 상황 | 현재 처리 |
| --- | --- |
| HTTP 429·5xx 또는 `HttpTimeoutException` | 같은 이력에 재시도 표식을 남기고, 상위 처리에 예외를 전달해 SQS 재전달을 유도한다. |
| 일반 I/O 오류·중단·응답 파싱 실패 | `FAILED / EXPO_REQUEST_UNCONFIRMED`로 기록한다. |
| Ticket 결과 개수 불일치 | `FAILED / EXPO_TICKET_RESULT_MISMATCH`로 기록한다. |
| 정상 응답 안의 Ticket 오류 | 해당 이력만 실패로 기록한다. |
| Ticket 저장 후 Receipt 예약 실패 | 재전달 때 접수 이력의 Receipt 확인을 다시 예약한다. Expo에 다시 보내지 않는다. |
| 준비·처리 중 예외 발생 | 아직 Expo에 넘기지 않은 대기 이력만 재시도 가능하게 복구한다. 이미 제출한 묶음을 일괄 재시도 대상으로 바꾸지 않는다. |

**timeout을 재시도 가능으로 분류하는 것은 현재 구현의 정책이지, Expo가 요청을 받지 않았다는 증거가 아니다.** 이미 접수했지만 응답이 늦어진 경우에는 재시도로 중복될 가능성이 남는다. 외부 접수와 DB 저장 사이의 실패도 있으므로, DB 키만으로 종단 간 exactly-once 발송을 보장한다고 설명하지 않는다.

표식 없는 `REQUESTED` 이력은 시간이 지났다는 이유만으로 재발송하지 않는다. 외부 접수 여부가 불확실한 이력과 아직 제출하지 않은 이력을 구분해서 조사해야 한다.

### 관리자 공지는 별도 정책이다

`sendAdminPrepared`는 `RetryablePushNotificationException`을 포함한 Push 요청 예외를 `EXPO_REQUEST_UNCONFIRMED`로 종료한다. 일반 발송의 429·5xx·timeout 재시도 정책을 관리자 공지에 그대로 적용하면 안 된다.

## Receipt 확인과 SQS 재전달의 차이

Ticket은 요청 접수 결과이고 Receipt는 이후 전달 처리 결과다. Receipt 예약의 기본 지연은 900초다. 설정값과 Queue 대기 시간에 따라 실제 확인 시각은 달라질 수 있다.

- Receipt가 아직 없으면 `attempt` 1 → 2 → 3으로 확인을 예약한다. 세 번째에도 없으면 `ReceiptNotAvailable`로 실패 처리한다.
- 이 숫자는 **Receipt 미준비에 대한 논리적인 확인 단계**다. SQS의 동일 메시지 재전달 횟수와 다르며, 실제 HTTP 호출이 무조건 총 3회라는 뜻은 아니다.
- SQS 메시지는 처리 성공 후 삭제하며, 처리 실패는 Queue 재전달·DLQ 정책을 따른다. 수동 재처리는 접수 이력과 외부 결과를 확인한 뒤 범위를 정해야 한다.

## 코드와 검증 근거

아래 테스트는 정책의 기존 검증 위치다. 실제 기기 수신이나 운영 처리 시간은 별도로 확인해야 한다.

| 확인할 내용 | 기준 코드·테스트 |
| --- | --- |
| 정기 이벤트 키 | [ScheduledNotificationService](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/notification/scheduled/service/ScheduledNotificationService.java.md) |
| 일반·관리자 발송 차이와 실패 복구 | [NotificationDispatchService](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/notification/delivery/service/NotificationDispatchService.java.md), [테스트](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/test/java/com/landit/landitbe/feature/notification/delivery/service/NotificationDispatchServiceTest.java.md) |
| 선점·상태 전이 | [PushDeliveryService](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/notification/delivery/service/PushDeliveryService.java.md), [재시도 동시성 테스트](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/test/java/com/landit/landitbe/feature/notification/delivery/domain/PushDeliveryRetryConcurrencyIntegrationTests.java.md) |
| 외부 오류 분류 | [ExpoPushClient](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/notification/delivery/client/expo/ExpoPushClient.java.md) |
| Receipt 단계·예약 | [PushReceiptService](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/notification/delivery/service/PushReceiptService.java.md), [SqsPushQueuePublisher](https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/notification/delivery/messaging/SqsPushQueuePublisher.java.md) |
