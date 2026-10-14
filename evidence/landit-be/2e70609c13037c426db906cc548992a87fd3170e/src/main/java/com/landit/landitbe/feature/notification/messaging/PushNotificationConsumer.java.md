# PushNotificationConsumer.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `2e70609c13037c426db906cc548992a87fd3170e`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/notification/messaging/PushNotificationConsumer.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–45행


````java
// Push 전용 SQS 메시지를 동시성 2로 소비해 Handler에 전달한다.

package com.landit.landitbe.feature.notification.messaging;

import io.awspring.cloud.sqs.annotation.SqsListener;
import io.awspring.cloud.sqs.listener.Visibility;
import lombok.RequiredArgsConstructor;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.stereotype.Component;

/** Push 전용 SQS 메시지를 동시성 2로 소비해 Handler에 전달한다. */
@Component
@RequiredArgsConstructor
@ConditionalOnProperty(
    prefix = "landit.notification",
    name = "consumer-enabled",
    havingValue = "true")
public class PushNotificationConsumer {

  private final PushQueueMessageHandler pushQueueMessageHandler;

  /**
   * Push Queue 메시지를 처리하고 성공한 경우에만 SQS에서 삭제되도록 반환한다.
   *
   * @param message Push Queue 메시지
   * @param visibility 현재 SQS 메시지의 visibility를 연장하는 객체
   */
  @SqsListener(
      value = "${landit.notification.queue-url}",
      maxConcurrentMessages = "2",
      maxMessagesPerPoll = "2",
      acknowledgementMode = "ON_SUCCESS")
  public void consume(PushQueueMessage message, Visibility visibility) {
    pushQueueMessageHandler.handle(message, () -> visibility.changeTo(300));
  }

  /**
   * 단위 테스트와 직접 호출에서 SQS visibility 연장 없이 메시지를 처리한다.
   *
   * @param message Push Queue 메시지
   */
  public void consume(PushQueueMessage message) {
    pushQueueMessageHandler.handle(message);
  }
}
````
