# DailyRewardAssignmentService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `eac4317c00d454b327c6a3e5be482bcf56961297`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/learning/scenario/selection/service/DailyRewardAssignmentService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 24–42행

<a id="source-L32-L42"></a>

````java
  private final RewardAssignmentService assignments;
  private final CurrentScenarioSelectionService selection;
  private final ScenarioAccessService access;
  private final ScenarioLearningLevelService levels;
  private final ProfileLearningService profiles;
  private final ExpressionContentService expressions;
  private final Clock clock;

  /**
   * 결제·학습·마감 정산과 같은 트랜잭션에서 실제 학습 배정을 고정한다.
   *
   * <p>동기 이벤트로 완료해야 정산 금액 계산보다 먼저 배정이 확정된다.
   *
   * @param request 인증된 사용자와 정산 기준 시각
   */
  @org.springframework.context.event.EventListener
  @Transactional
  public void prepareRewardDay(com.landit.landitbe.feature.reward.dto.RewardDayRequested request) {
    capture(request.userId(), request.evaluatedAt());
````

## 원본 56–96행

<a id="source-L64-L96"></a>

````java
  }

  /**
   * 완료의 원본 시각과 같은 날짜에 배정을 고정한다.
   *
   * @param userId 사용자 ID
   * @param now 호출자가 한 번 고정한 서버 기준 시각
   */
  @Transactional
  public void capture(long userId, java.time.Instant now) {
    if (!properties.collecting()) {
      return;
    }
    var date = now.atZone(ZoneId.of("Asia/Seoul")).toLocalDate();
    if (assignments.find(userId, date).isPresent()) {
      return;
    }
    assignments.lockForCapture(userId);
    if (assignments.find(userId, date).isPresent()) {
      return;
    }
    var locale = profiles.getUserLocale(userId);
    Long scenarioId =
        access
            .findAccessGrantedOn(userId, locale.targetLocale(), date)
            .map(h -> h.scenarioId())
            .orElseGet(
                () ->
                    selection
                        .findCurrentScenario(userId, locale.targetLocale(), now)
                        .map(s -> s.scenarioId())
                        .orElse(null));
    List<Long> ids =
        scenarioId == null
            ? List.of()
            : expressions.findScenarioExpressionIds(
                scenarioId,
                locale.targetLocale(),
                locale.baseLocale(),
                levels.expressionLevel(userId, scenarioId));
    assignments.capture(userId, new RewardAssignment(date, scenarioId, ids));
````
