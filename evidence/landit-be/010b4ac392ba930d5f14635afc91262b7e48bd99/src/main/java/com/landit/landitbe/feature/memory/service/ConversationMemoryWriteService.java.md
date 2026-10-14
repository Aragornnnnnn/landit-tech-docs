# ConversationMemoryWriteService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `010b4ac392ba930d5f14635afc91262b7e48bd99`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/memory/service/ConversationMemoryWriteService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 47–171행

<a id="source-L117"></a>
<a id="source-L55"></a>
<a id="source-L95"></a>

````java
   * @param userProfileId 잠글 사용자 프로필 ID
   * @param plans 후보별 상태 판정과 비교 검색 snapshot
   * @return snapshot이 같아 저장을 완료했으면 STORED와 계획 순번별 새 기억 ID, 달라졌으면 STALE
   * @throws ApiException 활성 사용자 프로필이 없을 때
   * @throws IllegalArgumentException 후보 계획이 사용자 범위 또는 상태 계약에 맞지 않을 때
   * @throws IllegalStateException 대체 대상이 활성 상태가 아닐 때
   */
  @Transactional
  public ConversationMemoryPersistence persistIfSnapshotCurrent(
      long userProfileId, List<ConversationMemoryResolutionPlan> plans) {
    requirePositive(userProfileId, "사용자 프로필 ID");
    if (plans == null) {
      throw new IllegalArgumentException("장기기억 판정 계획이 필요합니다.");
    }

    profileLearningService
        .findActiveLearningProfileForUpdate(userProfileId)
        .orElseThrow(() -> new ApiException(ErrorCode.RESOURCE_NOT_FOUND));
    validatePlans(userProfileId, plans);
    if (snapshotChanged(plans)) {
      return new ConversationMemoryPersistence(PersistenceResult.STALE, Map.of());
    }

    Map<Integer, Long> savedMemoryIdsByPlanIndex = new HashMap<>();
    for (int planIndex = 0; planIndex < plans.size(); planIndex++) {
      Long savedMemoryId = persistPlan(plans.get(planIndex));
      if (savedMemoryId != null) {
        savedMemoryIdsByPlanIndex.put(planIndex, savedMemoryId);
      }
    }
    return new ConversationMemoryPersistence(PersistenceResult.STORED, savedMemoryIdsByPlanIndex);
  }

  /**
   * 기억 저장과 같은 트랜잭션에서, 저장을 마친 뒤에도 그 기억이 활성 상태인지 확인한다.
   *
   * <p>사용자 잠금을 쥔 저장 트랜잭션 안에서만 부른다. 이번 저장 계획이나 그사이 끝난 다른 작업이 그 기억을 대체했는지를 저장 이후 기준으로 알 수 있다.
   *
   * @param userProfileId 기억 소유 사용자 프로필 ID
   * @param memoryId 확인할 장기기억 ID
   * @return 본인 소유의 활성 기억이면 true
   */
  @Transactional(propagation = Propagation.MANDATORY)
  public boolean isActiveAfterPersistence(long userProfileId, long memoryId) {
    return memoryRepository.existsActive(userProfileId, memoryId);
  }

  /** 사용자 잠금 안에서 비교 목록을 다시 조회해 AI 판정 시점 이후 변경을 차단한다. */
  private boolean snapshotChanged(List<ConversationMemoryResolutionPlan> plans) {
    for (ConversationMemoryResolutionPlan plan : plans) {
      NewConversationMemory memory = plan.memory();
      List<Long> currentIds =
          searchRepository
              .searchActiveComparable(
                  memory.embedding(),
                  memory.userProfileId(),
                  memory.characterId(),
                  memory.memoryType(),
                  MAX_COMPARABLE_MEMORIES)
              .stream()
              .map(ConversationMemoryMatch::memoryId)
              .toList();
      if (!currentIds.equals(plan.snapshotMemoryIds())) {
        return true;
      }
    }
    return false;
  }

  /** 상태 판정 결과에 따라 신규 기억 저장과 기존 기억 대체를 순서대로 수행하고 새 기억 ID를 돌려준다. IGNORE면 null이다. */
  private Long persistPlan(ConversationMemoryResolutionPlan plan) {
    if (plan.operation() == AiMemoryOperation.IGNORE) {
      return null;
    }

    long newMemoryId = memoryRepository.save(plan.memory(), plan.sourceMessageIds());
    if (plan.operation() == AiMemoryOperation.ADD) {
      return newMemoryId;
    }

    LocalDateTime supersededAt = LocalDateTime.now();
    for (Long oldMemoryId : plan.supersededMemoryIds()) {
      if (!memoryRepository.supersedeActive(
          oldMemoryId, newMemoryId, plan.memory().validFrom(), supersededAt)) {
        throw new IllegalStateException("대체할 활성 장기기억이 없습니다.");
      }
    }
    return newMemoryId;
  }

  /** 저장 계획은 잠금한 사용자와 snapshot 범위 안에서만 실행할 수 있다. */
  private static void validatePlans(
      long userProfileId, List<ConversationMemoryResolutionPlan> plans) {
    for (ConversationMemoryResolutionPlan plan : plans) {
      validatePlan(userProfileId, plan);
    }
  }

  /** 한 계획이 잠금한 사용자와 operation별 대체 범위를 벗어나지 않는지 확인한다. */
  private static void validatePlan(long userProfileId, ConversationMemoryResolutionPlan plan) {
    if (plan == null) {
      throw new IllegalArgumentException("장기기억 판정 계획이 유효하지 않습니다.");
    }
    NewConversationMemory memory = plan.memory();
    if (memory.userProfileId() != userProfileId) {
      throw new IllegalArgumentException("장기기억 사용자 범위가 일치하지 않습니다.");
    }
    if (plan.operation() == AiMemoryOperation.SUPERSEDE && plan.supersededMemoryIds().isEmpty()) {
      throw new IllegalArgumentException("대체할 장기기억이 필요합니다.");
    }
    if (plan.operation() == AiMemoryOperation.SUPERSEDE
        && !plan.snapshotMemoryIds().containsAll(plan.supersededMemoryIds())) {
      throw new IllegalArgumentException("대체할 장기기억이 비교 검색 snapshot에 없습니다.");
    }
    if (plan.operation() != AiMemoryOperation.SUPERSEDE && !plan.supersededMemoryIds().isEmpty()) {
      throw new IllegalArgumentException("ADD·IGNORE에는 대체할 장기기억이 없어야 합니다.");
    }
  }

  private static void requirePositive(long value, String fieldName) {
    if (value <= 0) {
      throw new IllegalArgumentException(fieldName + "이(가) 유효하지 않습니다.");
    }
  }
}
````
