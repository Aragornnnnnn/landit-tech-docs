# RewardLearningService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `33ee6ceed257d9233f569f11184d0acaee3ec362`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/reward/service/RewardLearningService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–238행


````java
// 검증된 활동 완료를 저장하고 최초 응답을 멱등한 적립 영수증으로 반환한다.

package com.landit.landitbe.feature.reward.service;

import com.landit.landitbe.config.reward.RewardProperties;
import com.landit.landitbe.feature.profile.subscription.service.ProfileSubscriptionService;
import com.landit.landitbe.feature.reward.dto.RewardCompletion;
import com.landit.landitbe.feature.reward.dto.RewardLearningFact;
import com.landit.landitbe.feature.reward.repository.RewardActivityRepository;
import com.landit.landitbe.feature.reward.repository.RewardRepository;
import com.landit.landitbe.shared.exception.ApiException;
import com.landit.landitbe.shared.exception.ErrorCode;
import java.time.Clock;
import java.time.LocalDateTime;
import java.time.ZoneId;
import java.util.List;
import java.util.Optional;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/** 학습 사실과 적립은 같은 트랜잭션이며 최초 영수증은 재계산으로 변경하지 않는다. */
@Service
@RequiredArgsConstructor
public class RewardLearningService {
  private static final ZoneId KST = ZoneId.of("Asia/Seoul");
  private final RewardProperties properties;
  private final RewardAccountService accounts;
  private final RewardRepository repository;
  private final RewardActivityRepository activities;
  private final RewardSettlementService settlement;
  private final Clock clock;
  private final ProfileSubscriptionService profiles;

  /**
   * 스몰톡의 검증된 완료를 저장한다.
   *
   * @param userId 사용자 ID
   * @param activity SMALLTALK
   * @param sessionId 원본 세션 ID
   * @param completedAt 최초 서버 완료 시각
   * @return 신규 환급 회차의 영수증. 환급 대상 상품 이력이 없는 이용자는 null
   * @throws IllegalArgumentException 완료 유형이나 시각이 유효하지 않을 때
   */
  @Transactional
  public RewardCompletion record(
      long userId, String activity, long sessionId, LocalDateTime completedAt) {
    if (!"SMALLTALK".equals(activity) || completedAt == null) {
      throw new IllegalArgumentException("Unsupported learning completion");
    }
    return recordFact(
        userId,
        new RewardLearningFact(
            0,
            activity,
            Long.toString(sessionId),
            null,
            null,
            completedAt.atZone(KST).toInstant()));
  }

  /**
   * 실제 시나리오 ID를 검증된 완료에 보존하여 당일 배정과 비교한다.
   *
   * @param userId 사용자 ID
   * @param sessionId 원본 세션 ID
   * @param scenarioId 완료한 시나리오 ID
   * @param completedAt 최초 서버 완료 시각
   * @return 해당 완료의 환급 영수증. 비대상 이용자는 null
   */
  @Transactional
  public RewardCompletion recordScenario(
      long userId, long sessionId, long scenarioId, LocalDateTime completedAt) {
    return recordFact(
        userId,
        new RewardLearningFact(
            0,
            "SCENARIO",
            Long.toString(sessionId),
            scenarioId,
            null,
            completedAt.atZone(KST).toInstant()));
  }

  /**
   * 권한 업무가 검증한 표현 시도의 최초 완료만 저장한다.
   *
   * @param userId 사용자 ID
   * @param expressionId 완료한 표현 ID
   * @param attemptId 검증된 시도 ID
   * @param completedAt 그 시도의 최초 서버 완료 시각
   * @return 신규 환급 영수증. 구버전의 검증 근거가 없으면 null
   */
  @Transactional
  public RewardCompletion recordExpression(
      long userId, long expressionId, String attemptId, LocalDateTime completedAt) {
    if (attemptId == null || completedAt == null) {
      return null;
    }
    return recordFact(
        userId,
        new RewardLearningFact(
            0, "EXPRESSION", attemptId, null, expressionId, completedAt.atZone(KST).toInstant()));
  }

  private RewardCompletion recordFact(long userId, RewardLearningFact candidate) {
    if (!properties.collecting()) {
      return null;
    }
    accounts.lock(userId);
    Optional<RewardLearningFact> previous =
        activities.fact(userId, candidate.activityType(), candidate.sourceKey());
    if (previous.isEmpty()) {
      activities.insert(userId, candidate);
    }
    RewardLearningFact fact =
        activities.fact(userId, candidate.activityType(), candidate.sourceKey()).orElseThrow();
    var receipt = activities.receipt(fact.id());
    if (receipt.isPresent()) {
      return properties.enabled() && properties.configured() ? receipt.get() : null;
    }
    settlement.settle(userId);
    RewardCompletion computed = calculateReceipt(userId, fact);
    if (computed != null) {
      activities.saveReceipt(computed);
    }
    return computed;
  }

  /**
   * 스몰톡 재요청에 최초 영수증을 전달한다.
   *
   * @param userId 사용자 ID
   * @param activity 활동 유형
   * @param sourceId 세션 ID
   * @return 최초 영수증. 환급 대상 상품 이력이 없는 이용자 또는 미완료면 null
   */
  @Transactional(readOnly = true)
  public RewardCompletion receipt(long userId, String activity, long sourceId) {
    if (!properties.enabled() || !visible(userId)) {
      return null;
    }
    return activities
        .fact(userId, activity, Long.toString(sourceId))
        .flatMap(f -> activities.receipt(f.id()))
        .orElse(null);
  }

  /**
   * 대기 또는 지연 보정된 완료의 최신 상태를 조회한다.
   *
   * @param userId 인증된 소유자
   * @param completionId 완료 ID
   * @return 현재 계산 버전의 상태
   * @throws ApiException 소유한 환급 완료를 찾을 수 없을 때
   */
  @Transactional(
      readOnly = true,
      isolation = org.springframework.transaction.annotation.Isolation.REPEATABLE_READ)
  public RewardCompletion completion(long userId, long completionId) {
    if (!properties.enabled()) {
      throw new ApiException(ErrorCode.RESOURCE_NOT_FOUND);
    }
    RewardLearningFact fact =
        activities
            .ownedFact(userId, completionId)
            .orElseThrow(() -> new ApiException(ErrorCode.RESOURCE_NOT_FOUND));
    RewardCompletion result = calculateReceipt(userId, fact);
    if (result == null) {
      throw new ApiException(ErrorCode.RESOURCE_NOT_FOUND);
    }
    return result;
  }

  private boolean visible(long userId) {
    if (!properties.enabled() || !properties.configured()) {
      return false;
    }
    return !repository.readCycles(userId).isEmpty() || currentNewProduct(userId);
  }

  private boolean currentNewProduct(long userId) {
    var profile = profiles.getSubscription(userId);
    return profile != null
        && profile.store() != null
        && properties.isNewSnapshot(profile.store().name(), profile.productId());
  }

  private RewardCompletion calculateReceipt(long userId, RewardLearningFact fact) {
    if (!visible(userId)) {
      return null;
    }
    var cycle =
        repository.readCycles(userId).stream()
            .filter(
                c ->
                    !fact.completedAt().isBefore(c.startsAt())
                        && fact.completedAt().isBefore(c.endsAt()))
            .findFirst()
            .orElse(null);
    if (cycle == null && !currentNewProduct(userId)) {
      return null;
    }
    var date = fact.completedAt().atZone(KST).toLocalDate();
    Long earned =
        cycle == null || !cycle.eligibility().equals("ELIGIBLE")
            ? null
            : activities.earned(fact.id());
    var day =
        cycle == null
            ? null
            : repository.days(cycle.id()).stream()
                .filter(d -> d.date().equals(date))
                .findFirst()
                .orElse(null);
    String status = earned == null ? "PENDING" : earned > 0 ? "EARNED" : "NO_ADDITIONAL_REWARD";
    String reason =
        earned == null
            ? cycle == null ? "PAID_INTERVAL_PENDING" : cycle.eligibility()
            : earned == 0 ? "NO_REMAINING_ALLOCATION" : null;
    return new RewardCompletion(
        fact.id(),
        cycle == null ? null : cycle.id(),
        fact.activityType(),
        date,
        status,
        reason,
        earned,
        earned == null ? null : activities.balance(fact.id()),
        earned == null || day == null ? null : day.earnedWon(),
        earned == null || day == null ? null : day.maximumWon(),
        day != null && earned != null && day.fullyAchieved(),
        earned == null || day == null ? List.of() : day.activities(),
        clock.instant(),
        cycle == null ? null : cycle.version(),
        "KRW");
  }
}
````
