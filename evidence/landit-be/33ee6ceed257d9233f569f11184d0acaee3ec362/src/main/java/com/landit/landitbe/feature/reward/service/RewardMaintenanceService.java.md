# RewardMaintenanceService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `33ee6ceed257d9233f569f11184d0acaee3ec362`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/reward/service/RewardMaintenanceService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–63행


````java
// 재시작 이후에도 미연결 결제와 마감 기한이 지난 도전을 복구한다.

package com.landit.landitbe.feature.reward.service;

import com.landit.landitbe.config.reward.RewardProperties;
import com.landit.landitbe.feature.reward.repository.RewardRepository;
import java.time.Clock;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

/** 사용자별 트랜잭션으로 처리해 일부 실패가 전체 마감을 되돌리지 않게 한다. */
@Service
@ConditionalOnProperty(
    name = "landit.reward.maintenance-enabled",
    havingValue = "true",
    matchIfMissing = true)
@RequiredArgsConstructor
@Slf4j
public class RewardMaintenanceService {
  private final RewardProperties properties;
  private final RewardRepository repository;
  private final RewardBillingService billing;
  private final Clock clock;
  private final RewardSettlementService settlement;

  /** 미연결 사실과 마감 도전을 한 번에 최대 100개씩 복구한다. */
  @Scheduled(fixedDelayString = "${landit.reward.maintenance-delay-ms:60000}")
  public void recover() {
    if (!properties.collecting()) {
      return;
    }
    for (var fact : repository.unresolvedBillings()) {
      try {
        billing.receive(fact.fact());
      } catch (RuntimeException failure) {
        repository.touchAttempt(fact.id());
        log.warn("Reward billing recovery failed: {}", failure.getClass().getSimpleName());
      }
    }
    if (!properties.enabled() || !properties.configured()) {
      return;
    }
    for (long userId : repository.dueUsers(clock.instant())) {
      try {
        settlement.settle(userId);
      } catch (RuntimeException failure) {
        repository.deferDueUser(userId, clock.instant().plusSeconds(60));
        log.warn("Reward settlement recovery failed: {}", failure.getClass().getSimpleName());
      }
    }
    for (long userId : repository.pendingUsers()) {
      try {
        billing.replayUser(userId);
      } catch (RuntimeException failure) {
        repository.touchPendingUser(userId);
        log.warn("Reward pending recovery failed: {}", failure.getClass().getSimpleName());
      }
    }
  }
}
````
