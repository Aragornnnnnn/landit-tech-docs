# RewardBillingService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `33ee6ceed257d9233f569f11184d0acaee3ec362`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/reward/service/RewardBillingService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–357행


````java
// 결제 원본을 수신하고 유료 회차 및 검토 상태를 멱등하게 조율한다.

package com.landit.landitbe.feature.reward.service;

import com.landit.landitbe.config.reward.RewardProperties;
import com.landit.landitbe.feature.profile.service.UserProfileService;
import com.landit.landitbe.feature.profile.subscription.service.ProfileSubscriptionService;
import com.landit.landitbe.feature.reward.domain.RewardCalculator;
import com.landit.landitbe.feature.reward.dto.RewardBillingFact;
import com.landit.landitbe.feature.reward.repository.RewardRepository;
import com.landit.landitbe.feature.reward.repository.RewardRepository.Billing;
import com.landit.landitbe.feature.reward.repository.RewardRepository.Cycle;
import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.HexFormat;
import java.util.List;
import java.util.Objects;
import java.util.Optional;
import lombok.RequiredArgsConstructor;
import org.springframework.core.env.Environment;
import org.springframework.core.env.Profiles;
import org.springframework.dao.DuplicateKeyException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/** 최신 구독 상태의 역순 이벤트 차단과 독립적으로 과거 결제 회차를 복구한다. */
@Service
@RequiredArgsConstructor
public class RewardBillingService {
  private final RewardProperties properties;
  private final RewardInboxService inbox;
  private final RewardRepository repository;
  private final RewardAccountService accounts;
  private final RewardSettlementService settlement;
  private final UserProfileService profiles;
  private final ProfileSubscriptionService subscriptionProfiles;
  private final Environment environment;

  /**
   * 인증 및 환경 검사를 통과한 제공자 사실을 저장하고 처리한다.
   *
   * @param fact 최소 결제 사실
   */
  @Transactional
  public void receive(RewardBillingFact fact) {
    if (!properties.collecting()) {
      return;
    }
    try {
      inbox.receive(fact);
    } catch (DuplicateKeyException duplicate) {
      if (repository.billing(fact).isEmpty()) {
        throw duplicate;
      }
    }
    process(repository.billing(fact).orElseThrow());
  }

  /**
   * 사용자에게 연결된 지연 사실을 재처리하고 도전을 마감한다.
   *
   * @param userId 사용자 ID
   */
  @Transactional
  public void replayUser(long userId) {
    List<Billing> facts = repository.billings(userId);
    var owners = new java.util.TreeSet<Long>();
    owners.add(userId);
    for (Billing item : facts) {
      owners.addAll(repository.transactionOwners(item.fact()));
    }
    lockOwners(owners);
    for (Billing item : facts) {
      purchase(userId, item);
    }
    reconcileOwners(owners);
  }

  private void lockOwners(java.util.SortedSet<Long> owners) {
    for (long owner : owners) {
      accounts.lock(owner);
    }
  }

  private void reconcileOwners(java.util.SortedSet<Long> owners) {
    for (long owner : owners) {
      reconcile(owner);
      settlement.settle(owner);
    }
  }

  private void process(Billing billing) {
    RewardBillingFact fact = billing.fact();
    if ("TRANSFER".equals(fact.type())
        && (fact.environment() == null
            || fact.environment().isBlank()
            || acceptedEnvironment(fact.environment()))) {
      transfer(billing);
      return;
    }
    if (!acceptedEnvironment(fact.environment())) {
      repository.markBilling(billing.id(), null, "IGNORED", "NON_PRODUCTION");
      return;
    }
    Optional<Long> resolved = resolve(fact.userCandidates());
    if (resolved.isEmpty()) {
      repository.markBilling(billing.id(), null, "PENDING", "USER_UNRESOLVED");
      return;
    }
    long userId = resolved.get();
    var owners = new java.util.TreeSet<Long>(repository.transactionOwners(fact));
    owners.add(userId);
    // 같은 거래의 기존 소유자까지 순서대로 잠가 조회 재처리 없이 검토 상태를 확정한다.
    lockOwners(owners);
    repository.markBilling(billing.id(), userId, "STORED", null);
    // 구매를 먼저 생성한 뒤 역순으로 수신한 예외 사실까지 대조한다.
    purchase(userId, billing);
    reconcileOwners(owners);
  }

  private void purchase(long userId, Billing billing) {
    RewardBillingFact fact = billing.fact();
    if (!acceptedEnvironment(fact.environment())) {
      return;
    }
    if (!"INITIAL_PURCHASE".equals(fact.type()) && !"RENEWAL".equals(fact.type())) {
      return;
    }
    if (fact.appId() != null
        && fact.environment() != null
        && fact.store() != null
        && fact.transactionId() != null
        && fact.startsAt() != null) {
      var existing = repository.cycleByKey(key(fact));
      if (existing.isPresent()) {
        reuse(userId, billing, existing.get());
        return;
      }
    }
    var product =
        properties.products() == null
            ? Optional.<RewardProperties.Product>empty()
            : properties.products().stream()
                .filter(Objects::nonNull)
                .filter(
                    p ->
                        Objects.equals(p.appId(), fact.appId())
                            && Objects.equals(p.store(), fact.store())
                            && Objects.equals(p.productId(), fact.productId()))
                .findFirst();
    if (product.isEmpty()) {
      repository.markBilling(billing.id(), userId, "UNMAPPED", "UNKNOWN_PRODUCT");
      return;
    }
    if ("TRIAL".equals(fact.periodType())) {
      repository.markBilling(billing.id(), userId, "IGNORED", "TRIAL_NOT_SUPPORTED");
      return;
    }
    if (!properties.enabled() || !properties.configured()) {
      repository.markBilling(billing.id(), userId, "PENDING", "CONFIGURATION_INCOMPLETE");
      return;
    }
    String invalid = validate(fact, product.get());
    if (invalid != null) {
      if (fact.transactionId() != null && fact.startsAt() != null) {
        repository
            .cycleByKey(key(fact))
            .filter(c -> c.userId() == userId)
            .ifPresent(c -> repository.review(c.id(), "TRANSACTION_CHANGED"));
      }
      repository.markBilling(billing.id(), userId, "REVIEW_REQUIRED", invalid);
      return;
    }
    int percent = product.get().months() == 3 ? 80 : 100;
    long maximum = fact.price().longValueExact() * percent / 100;
    repository.insertCycle(
        userId,
        key(fact),
        fact,
        percent,
        maximum,
        RewardCalculator.days(fact.startsAt(), fact.endsAt()),
        accounts.lock(userId).eligibility());
    repository.markBilling(billing.id(), userId, "PROCESSED", null);
  }

  private void reuse(long userId, Billing billing, Cycle cycle) {
    if (cycle.userId() != userId) {
      repository.reviewAccount(userId, "OWNER_CONFLICT");
      repository.markBilling(billing.id(), userId, "REVIEW_REQUIRED", "OWNER_CONFLICT");
      return;
    }
    RewardBillingFact fact = billing.fact();
    boolean same =
        cycle.endsAt().equals(fact.endsAt())
            && cycle.productId().equals(fact.productId())
            && fact.price() != null
            && fact.price().compareTo(java.math.BigDecimal.valueOf(cycle.priceWon())) == 0
            && "KRW".equals(fact.currency())
            && "NORMAL".equals(fact.periodType())
            && Boolean.FALSE.equals(fact.familyShare());
    if (!same) {
      repository.review(cycle.id(), "TRANSACTION_CHANGED");
    }
    if (same && "PROCESSED".equals(billing.status())) {
      return;
    }
    repository.markBilling(
        billing.id(),
        userId,
        same ? "PROCESSED" : "REVIEW_REQUIRED",
        same ? null : "TRANSACTION_CHANGED");
  }

  private boolean acceptedEnvironment(String billingEnvironment) {
    return "PRODUCTION".equals(billingEnvironment)
        || ("SANDBOX".equals(billingEnvironment)
            && environment.acceptsProfiles(Profiles.of("develop & !prod")));
  }

  private String validate(RewardBillingFact fact, RewardProperties.Product product) {
    if (fact.transactionId() == null
        || fact.transactionId().isBlank()
        || fact.startsAt() == null
        || fact.endsAt() == null) {
      return "MISSING_TRANSACTION";
    }
    if (!Boolean.FALSE.equals(fact.familyShare())) {
      return "FAMILY_SHARE_REQUIRES_REVIEW";
    }
    if (!"NORMAL".equals(fact.periodType())
        || !"KRW".equals(fact.currency())
        || fact.price() == null
        || fact.price()
                .compareTo(java.math.BigDecimal.valueOf(product.months() == 3 ? 39900 : 59900))
            != 0) {
      return "UNSUPPORTED_PRICE_OR_PERIOD";
    }
    try {
      RewardCalculator.days(fact.startsAt(), fact.endsAt());
    } catch (IllegalArgumentException invalid) {
      return "INVALID_INTERVAL";
    }
    return null;
  }

  private void reconcile(long userId) {
    List<Cycle> cycles = repository.cycles(userId);
    if (accounts.lock(userId).reviewReason() != null) {
      for (Cycle cycle : cycles) {
        repository.review(cycle.id(), accounts.lock(userId).reviewReason());
      }
    }
    for (int i = 0; i < cycles.size(); i++) {
      Cycle cycle = cycles.get(i);
      for (int j = i + 1; j < cycles.size(); j++) {
        Cycle other = cycles.get(j);
        if (cycle.startsAt().isBefore(other.endsAt())
            && other.startsAt().isBefore(cycle.endsAt())) {
          repository.review(cycle.id(), "OVERLAPPING_PAID_INTERVALS");
          repository.review(other.id(), "OVERLAPPING_PAID_INTERVALS");
        }
      }
      for (Billing billing : repository.transactionFacts(cycle)) {
        RewardBillingFact fact = billing.fact();
        if (billing.userId() != null
            && billing.userId() != userId
            && ("INITIAL_PURCHASE".equals(fact.type()) || "RENEWAL".equals(fact.type()))) {
          repository.review(cycle.id(), "OWNER_CONFLICT");
        }
        boolean unknownCancellation =
            "CANCELLATION".equals(fact.type())
                && !List.of("UNSUBSCRIBE", "BILLING_ERROR", "CUSTOMER_SUPPORT")
                    .contains(fact.reason() == null ? "" : fact.reason());
        boolean exception =
            unknownCancellation
                || ("CANCELLATION".equals(fact.type()) && "CUSTOMER_SUPPORT".equals(fact.reason()))
                || ("EXPIRATION".equals(fact.type())
                    && fact.endsAt() != null
                    && fact.endsAt().isBefore(cycle.endsAt()))
                || "SUBSCRIPTION_EXTENDED".equals(fact.type())
                || "REFUND_REVERSED".equals(fact.type());
        if (exception) {
          repository.review(cycle.id(), "PAYMENT_EXCEPTION");
        }
      }
    }
  }

  private void transfer(Billing billing) {
    List<Long> users =
        billing.fact().userCandidates().stream()
            .map(this::resolveCandidate)
            .flatMap(Optional::stream)
            .distinct()
            .sorted()
            .toList();
    // 이전 대상의 프로필을 모두 먼저 잠근 뒤 보상 잠금에 접근한다.
    for (long userId : users) {
      subscriptionProfiles.lockRewardProfile(userId);
    }
    for (long userId : users) {
      accounts.lock(userId);
    }
    for (long userId : users) {
      repository.reviewAccount(userId, "TRANSFER_REQUIRES_REVIEW");
      for (Cycle cycle : repository.cycles(userId)) {
        repository.review(cycle.id(), "TRANSFER_REQUIRES_REVIEW");
      }
      settlement.settle(userId);
    }
    repository.markBilling(
        billing.id(),
        null,
        users.isEmpty() ? "PENDING" : "REVIEW_REQUIRED",
        "TRANSFER_REQUIRES_REVIEW");
  }

  private Optional<Long> resolve(List<String> candidates) {
    for (String candidate : candidates) {
      var user = resolveCandidate(candidate);
      if (user.isPresent()) {
        return user;
      }
    }
    return Optional.empty();
  }

  private Optional<Long> resolveCandidate(String candidate) {
    try {
      return profiles.findExistingUserId(List.of(Long.parseLong(candidate)));
    } catch (NumberFormatException anonymous) {
      return Optional.empty();
    }
  }

  private String key(RewardBillingFact fact) {
    try {
      String raw =
          List.of(
                  fact.appId(),
                  fact.environment(),
                  fact.store(),
                  fact.transactionId(),
                  fact.startsAt().toString())
              .stream()
              .map(s -> s.length() + ":" + s)
              .collect(java.util.stream.Collectors.joining());
      return HexFormat.of()
          .formatHex(
              MessageDigest.getInstance("SHA-256").digest(raw.getBytes(StandardCharsets.UTF_8)));
    } catch (NoSuchAlgorithmException impossible) {
      throw new IllegalStateException(impossible);
    }
  }
}
````
