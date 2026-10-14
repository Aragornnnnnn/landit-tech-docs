# SubscriptionLaunchPolicyService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `cfb15ec066277ed0baa255395c02c6c0500e99d7`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/subscription/service/SubscriptionLaunchPolicyService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–53행


````java
// 기존 SSM 오픈 시각으로 구독 제한의 적용 여부를 판단한다.

package com.landit.landitbe.feature.subscription.service;

import com.landit.landitbe.config.subscription.SubscriptionProperties;
import com.landit.landitbe.feature.subscription.dto.SubscriptionLaunchPolicy;
import java.time.Clock;
import java.time.LocalDateTime;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

/** 결제 공개 제어를 기존 환경변수 설정에 유지한다. */
@Service
@RequiredArgsConstructor
public class SubscriptionLaunchPolicyService {
  private final SubscriptionProperties properties;
  private final Clock clock;

  /**
   * 서버 시작 시 SSM에서 주입한 오픈 시각을 사용한다. DB에는 공개 스위치를 저장하지 않는다.
   *
   * @return 기존 설정의 공개 정책. 버전과 새 시작 중지는 이번 범위에서 사용하지 않는다.
   */
  public SubscriptionLaunchPolicy current() {
    LocalDateTime at =
        properties
            .launchedAtOrEmpty()
            .map(value -> value.atZoneSameInstant(clock.getZone()).toLocalDateTime())
            .orElse(null);
    return new SubscriptionLaunchPolicy(0, at, false);
  }

  /**
   * 기존 SSM 정책은 모든 사용자에게 같은 오픈 시각을 적용한다.
   *
   * @param policy 같은 요청에서 조회한 공개 정책
   * @param userId 학습 사용자 ID
   * @return 실제 활성 시각에 도달했으면 true
   */
  public boolean enabledFor(SubscriptionLaunchPolicy policy, long userId) {
    return active(policy);
  }

  /**
   * 오픈 시각이 없거나 아직 미래라면 유료 제한을 적용하지 않는다.
   *
   * @param policy 같은 요청에서 조회한 공개 정책
   * @return 실제 활성 시각에 도달했으면 true
   */
  public boolean active(SubscriptionLaunchPolicy policy) {
    return policy.effectiveAt() != null && !LocalDateTime.now(clock).isBefore(policy.effectiveAt());
  }
}
````
