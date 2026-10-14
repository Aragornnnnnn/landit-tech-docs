# RewardPostgresIntegrationTest.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `eac4317c00d454b327c6a3e5be482bcf56961297`.
- 원본 경로: `src/test/java/com/landit/landitbe/feature/reward/RewardPostgresIntegrationTest.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 176–313행

<a id="source-L184-L243"></a>
<a id="source-L247-L260"></a>
<a id="source-L264-L313"></a>

````java
            new com.landit.landitbe.feature.reward.dto.RewardAssignment(
                completedAt.toLocalDate(), 1L, List.of()));
    return activity.equals("SCENARIO")
        ? learning.recordScenario(userId, sourceId, 1L, completedAt)
        : learning.record(userId, activity, sourceId, completedAt);
  }

  @Test
  void firstScenarioAssignmentCommitsBeforeResponseAssembly() throws Exception {
    var assignments = context.getBean(RewardAssignmentService.class);
    var learningProfiles = mock(ProfileLearningService.class);
    when(learningProfiles.getUserLocale(1L)).thenReturn(new UserLocale(Locale.EN, Locale.KR));
    context.registerBean(
        DailyRewardAssignmentService.class,
        () ->
            new DailyRewardAssignmentService(
                context.getBean(RewardProperties.class),
                assignments,
                mock(CurrentScenarioSelectionService.class),
                mock(ScenarioAccessService.class),
                mock(ScenarioLearningLevelService.class),
                learningProfiles,
                mock(ExpressionContentService.class),
                context.getBean(Clock.class)));
    var daily = context.getBean(DailyRewardAssignmentService.class);
    var scenarioReads = mock(DailyScenarioReadService.class);
    var today = LocalDateTime.ofInstant(START, KST).toLocalDate();
    when(scenarioReads.getDailyScenario(1L, today, START))
        .thenAnswer(
            call -> {
              assertProfileLockAvailable();
              return DailyScenarioResponse.empty(today);
            });
    context.registerBean(
        DailyScenarioQueryService.class,
        () -> new DailyScenarioQueryService(daily, scenarioReads, context.getBean(Clock.class)));
    var scenarios = context.getBean(DailyScenarioQueryService.class);
    scenarios.getDailyScenario(1L, null);
    assertThat(assignments.find(1L, today)).isPresent();
  }

  private void assertProfileLockAvailable() throws Exception {
    try (var executor = Executors.newSingleThreadExecutor()) {
      executor
          .submit(
              () ->
                  new TransactionTemplate(context.getBean(PlatformTransactionManager.class))
                      .executeWithoutResult(
                          status -> {
                            jdbc.execute("SET LOCAL lock_timeout='1s'");
                            context.getBean(RewardAccountService.class).lock(1L);
                          }))
          .get(3, TimeUnit.SECONDS);
    }
  }

  @Test
  void firstConcurrentAssignmentIsNeverOverwritten() throws Exception {
    var assignments = context.getBean(RewardAssignmentService.class);
    var date = LocalDateTime.ofInstant(START, KST).toLocalDate();
    parallel(
        () -> assignments.capture(1L, new RewardAssignment(date, 10L, List.of(1L))),
        () -> assignments.capture(1L, new RewardAssignment(date, 20L, List.of(2L))));
    var original = assignments.find(1L, date).orElseThrow();
    assignments.capture(1L, new RewardAssignment(date, 30L, List.of(3L)));
    assertThat(assignments.find(1L, date)).contains(original);
    assertThat(jdbc.queryForObject("SELECT COUNT(*) FROM reward_daily_assignment", Integer.class))
        .isEqualTo(1);
  }

  @Test
  void readKeepsStoredResultsUntilScheduledDeadlineSettlement() {
    billing.receive(fact(1L));
    recordConversation(1L, "SCENARIO", 100L, LocalDateTime.ofInstant(START, KST));
    now.set(Instant.parse("2026-10-08T15:00:00Z"));
    final var versions = rewardRowVersions();
    assertThat(query.summary(1L).current().balanceWon()).isEqualTo(171);
    assertThat(query.calendar(1L, java.time.YearMonth.of(2026, 10)).summary().todayMaximumWon())
        .isNull();
    assertThat(rewardRowVersions()).isEqualTo(versions);
    context.getBean(RewardMaintenanceService.class).recover();
    assertThat(query.summary(1L).current().balanceWon()).isZero();
    assertThat(query.summary(1L).lostYesterdayWon()).isEqualTo(171);
    assertThat(query.calendar(1L, java.time.YearMonth.of(2026, 10)).summary().todayMaximumWon())
        .isEqualTo(343);
  }

  @Test
  void readEndpointsDoNotWaitForWriterLocksOrModifySnapshots() throws Exception {
    billing.receive(fact(1L));
    var receipt = recordConversation(1L, "SCENARIO", 100L, LocalDateTime.ofInstant(START, KST));
    long cycle = receipt.cycleId();
    final var versions = rewardRowVersions();
    var executor = Executors.newSingleThreadExecutor();
    try (var connection = context.getBean(DataSource.class).getConnection()) {
      connection.setAutoCommit(false);
      try (var statement = connection.createStatement()) {
        statement.execute("SELECT * FROM user_profile WHERE id=1 FOR UPDATE");
        statement.execute("SELECT * FROM reward_account WHERE user_id=1 FOR UPDATE");
        statement.execute("SELECT * FROM reward_cycle WHERE user_id=1 FOR UPDATE");
      }
      try {
        executor
            .submit(
                () -> {
                  assertThat(query.summary(1L).current().balanceWon()).isEqualTo(171);
                  assertThat(query.page(1L, null, 10).cycles()).hasSize(1);
                  assertThat(query.detail(1L, cycle).balanceWon()).isEqualTo(171);
                  assertThat(
                          query
                              .calendar(1L, java.time.YearMonth.of(2026, 10))
                              .summary()
                              .currentBalanceWon())
                      .isEqualTo(171);
                  assertThat(learning.completion(1L, receipt.completionId()).earnedWon())
                      .isEqualTo(171);
                  assertThat(
                          context
                              .getBean(
                                  com.landit.landitbe.feature.reward.service.RewardHistoryService
                                      .class)
                              .page(1L, null, 40)
                              .items())
                      .hasSize(1);
                })
            .get(5, TimeUnit.SECONDS);
      } finally {
        connection.rollback();
      }
    } finally {
      executor.shutdownNow();
    }
    assertThat(rewardRowVersions()).isEqualTo(versions);
    query.summary(2L);
    assertThat(
            jdbc.queryForObject(
                "SELECT COUNT(*) FROM reward_account WHERE user_id=2", Integer.class))
        .isZero();
````

## 원본 575–601행

<a id="source-L583-L601"></a>

````java
    assertThat(jdbc.queryForObject("SELECT COUNT(*) FROM reward_billing_event", Integer.class))
        .isEqualTo(1);
    assertThat(
            jdbc.queryForObject(
                "SELECT SUM(delta_won) FROM reward_ledger WHERE cycle_id=?", Long.class, cycle))
        .isEqualTo(343);
  }

  @Test
  void failedCalculationKeepsReceiptAndRestartReplaysIt() {
    var fact = fact(1L);
    var transaction = new TransactionTemplate(context.getBean(PlatformTransactionManager.class));
    assertThatThrownBy(
            () ->
                transaction.executeWithoutResult(
                    status -> {
                      billing.receive(fact);
                      throw new IllegalStateException("simulated process failure");
                    }))
        .isInstanceOf(IllegalStateException.class);
    assertThat(jdbc.queryForObject("SELECT COUNT(*) FROM reward_cycle", Integer.class)).isZero();
    assertThat(jdbc.queryForObject("SELECT COUNT(*) FROM reward_billing_event", Integer.class))
        .isEqualTo(1);
    context.getBean(RewardMaintenanceService.class).recover();
    assertThat(jdbc.queryForObject("SELECT COUNT(*) FROM reward_cycle", Integer.class))
        .isEqualTo(1);
    assertThat(query.summary(1L).current().eligibleDays()).isEqualTo(93);
````
