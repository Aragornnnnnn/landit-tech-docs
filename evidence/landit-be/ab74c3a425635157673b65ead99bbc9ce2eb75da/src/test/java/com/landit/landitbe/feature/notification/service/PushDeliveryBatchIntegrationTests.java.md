# PushDeliveryBatchIntegrationTests.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `ab74c3a425635157673b65ead99bbc9ce2eb75da`.
- 원본 경로: `src/test/java/com/landit/landitbe/feature/notification/service/PushDeliveryBatchIntegrationTests.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 158–362행


````java

  /** 소유자 변경·해제 Token은 제외하고 재시도 표식은 한 번만 소비한다. */
  @Test
  void preservesOwnerRevocationAndRetryGuards() {
    List<UserPushToken> seeded = seed(3);
    List<PreparePushDeliveryCommand> commands =
        seeded.stream().map(t -> command(t, "event")).toList();
    List<PreparedPushDelivery> prepared = deliveries.prepareAll(commands);
    prepared.forEach(p -> deliveries.markRetryable(p.pushDeliveryId()));
    seeded.get(0).claim(900L, AppPlatform.IOS);
    seeded.get(1).revoke();
    tokens.saveAllAndFlush(seeded);
    List<PreparedPushDelivery> retry = deliveries.prepareAll(commands);
    assertThat(retry).containsExactly(prepared.get(2));
    assertThat(deliveries.prepareAll(commands)).isEmpty();
    assertThat(
            deliveries.prepareAll(
                seeded.stream()
                    .map(t -> commandWithOwner(t, "new", 1L + seeded.indexOf(t)))
                    .toList()))
        .extracting(PreparedPushDelivery::expoPushToken)
        .containsExactly(seeded.get(2).getExpoPushToken());
  }

  /** 묶음 중 잘못된 Ticket이 있으면 앞선 전이도 롤백한다. */
  @Test
  void rollsBackWholeTicketBatchOnInvalidResult() {
    List<Long> ids =
        deliveries.prepareAll(seed(2).stream().map(t -> command(t, "event")).toList()).stream()
            .map(PreparedPushDelivery::pushDeliveryId)
            .toList();
    assertThatThrownBy(
            () ->
                deliveries.recordTicketResults(
                    ids,
                    List.of(
                        PushTicketResult.accepted("valid"), new PushTicketResult(true, "", null))))
        .isInstanceOf(IllegalArgumentException.class);
    assertThat(repository.findAll())
        .allSatisfy(d -> assertThat(d.getStatus()).isEqualTo(PushDeliveryStatus.REQUESTED));
  }

  /** 사용자 두 명의 발송 선점 중 한 행이 실패하면 INSERT 전체가 롤백된다. */
  @Test
  void rollsBackWholeInsertBatch() {
    List<UserPushToken> seeded = seed(2);
    PreparePushDeliveryCommand valid = command(seeded.getFirst(), "event");
    PreparePushDeliveryCommand invalid =
        new PreparePushDeliveryCommand(
            2L,
            seeded.getLast().getId(),
            NotificationType.CONTINUE_EXPRESSION,
            "bad",
            "x".repeat(256),
            "body",
            "/home");
    assertThatThrownBy(() -> deliveries.prepareAll(List.of(valid, invalid)))
        .isInstanceOf(RuntimeException.class);
    assertThat(repository.count()).isZero();
  }

  /** JDBC 저장도 외부 Spring 트랜잭션의 롤백에 함께 참여한다. */
  @Test
  void rollsBackJdbcInsertAndUpdateWithEnclosingTransaction() {
    UserPushToken token = seed(1).getFirst();
    assertThatThrownBy(
            () ->
                new TransactionTemplate(transactionManager)
                    .executeWithoutResult(
                        status -> {
                          PreparedPushDelivery prepared =
                              deliveries.prepareAll(List.of(command(token, "event"))).getFirst();
                          deliveries.recordTicketResults(
                              List.of(prepared.pushDeliveryId()),
                              List.of(PushTicketResult.accepted("ticket")));
                          throw new IllegalStateException("rollback after JDBC writes");
                        }))
        .isInstanceOf(IllegalStateException.class)
        .hasMessage("rollback after JDBC writes");
    assertThat(repository.count()).isZero();
  }

  /** Receipt 예약 실패 뒤 Token이 해제돼도 접수 이력으로 복구하며 Expo에 다시 보내지 않는다. */
  @Test
  void recoversAcceptedReceiptsAfterPublisherFailureWithoutResending() {
    List<UserPushToken> seeded = seed(2);
    NotificationSender sender = sender();
    PushQueuePublisher publisher = mock(PushQueuePublisher.class);
    doThrow(new PushNotificationException("partial failure"))
        .doNothing()
        .when(publisher)
        .scheduleReceiptChecks(anyList(), org.mockito.ArgumentMatchers.eq(1));
    NotificationDispatchService dispatch = dispatch(sender, publisher);
    List<SendPushNotificationCommand> commands =
        seeded.stream().map(t -> sendCommand(t, "event")).toList();
    assertThatThrownBy(() -> dispatch.sendAll(commands))
        .isInstanceOf(PushNotificationException.class);
    assertThat(repository.findAll())
        .allSatisfy(d -> assertThat(d.getStatus()).isEqualTo(PushDeliveryStatus.TICKET_ACCEPTED));
    seeded.forEach(UserPushToken::revoke);
    tokens.saveAllAndFlush(seeded);
    assertThat(dispatch.sendAll(commands).preparedDeliveries()).isZero();
    verify(sender, times(1)).send(anyList());
    verify(publisher, times(2))
        .scheduleReceiptChecks(anyList(), org.mockito.ArgumentMatchers.eq(1));
  }

  /** 접두어 안의 %, _를 패턴으로 해석하거나 다른 이벤트로 확대하지 않는다. */
  @Test
  void matchesEventPrefixesLiterally() {
    UserPushToken token = seed(1).getFirst();
    for (String event : List.of("a%_", "abc", "a%_extra")) {
      PreparedPushDelivery p = deliveries.prepareAll(List.of(command(token, event))).getFirst();
      deliveries.recordTicketResults(
          List.of(p.pushDeliveryId()), List.of(PushTicketResult.accepted(event)));
    }
    assertThat(deliveries.findAcceptedDeliveryIdsForEvents(List.of("push:a%_:1:"))).hasSize(1);
  }

  /** 같은 Token을 포함한 역순 신규 묶음은 이력을 한 번만 선점하고 교착되지 않는다. */
  @Test
  void claimsOverlappingNewBatchesOnlyOnce() throws Exception {
    List<PreparePushDeliveryCommand> commands =
        seed(3).stream().map(t -> command(t, "event")).toList();
    CountDownLatch start = new CountDownLatch(1);
    try (var executor = Executors.newFixedThreadPool(2)) {
      var first =
          executor.submit(
              () -> {
                await(start);
                return deliveries.prepareAll(commands);
              });
      var second =
          executor.submit(
              () -> {
                await(start);
                return deliveries.prepareAll(commands.reversed());
              });
      start.countDown();
      List<Long> ids = new ArrayList<>();
      first.get(10, TimeUnit.SECONDS).forEach(p -> ids.add(p.pushDeliveryId()));
      second.get(10, TimeUnit.SECONDS).forEach(p -> ids.add(p.pushDeliveryId()));
      assertThat(ids).hasSize(3).doesNotHaveDuplicates();
      assertThat(repository.count()).isEqualTo(3);
    }
  }

  /** 재시도 묶음이 잠금을 보유하면 기존 단건 경로도 기다리고 표식을 중복 소비하지 않는다. */
  @Test
  void serializesBatchRetryWithLegacySinglePrepare() throws Exception {
    PreparePushDeliveryCommand command = command(seed(1).getFirst(), "event");
    PreparedPushDelivery original = deliveries.prepareAll(List.of(command)).getFirst();
    deliveries.markRetryable(original.pushDeliveryId());
    CountDownLatch locked = new CountDownLatch(1);
    CountDownLatch commit = new CountDownLatch(1);
    CountDownLatch secondStarted = new CountDownLatch(1);
    try (var executor = Executors.newFixedThreadPool(2)) {
      var first =
          executor.submit(
              () ->
                  new TransactionTemplate(transactionManager)
                      .execute(
                          status -> {
                            List<PreparedPushDelivery> result =
                                deliveries.prepareAll(List.of(command));
                            locked.countDown();
                            await(commit);
                            return result;
                          }));
      try {
        assertThat(locked.await(5, TimeUnit.SECONDS)).isTrue();
        var second =
            executor.submit(
                () -> {
                  secondStarted.countDown();
                  return deliveries.prepare(command);
                });
        assertThat(secondStarted.await(5, TimeUnit.SECONDS)).isTrue();
        assertThatThrownBy(() -> second.get(200, TimeUnit.MILLISECONDS))
            .isInstanceOf(TimeoutException.class);
        commit.countDown();
        assertThat(first.get(5, TimeUnit.SECONDS)).containsExactly(original);
        assertThat(second.get(5, TimeUnit.SECONDS)).isEmpty();
      } finally {
        commit.countDown();
      }
    }
  }

  private void await(CountDownLatch latch) {
    try {
      if (!latch.await(5, TimeUnit.SECONDS)) {
        throw new IllegalStateException("동시성 테스트 제한시간 초과");
      }
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
      throw new IllegalStateException(e);
    }
  }

  /** Token 이전 트랜잭션이 먼저 잠갔다면 묶음 선점은 커밋된 새 소유자를 보고 제외한다. */
  @Test
  void waitsForTokenOwnershipChangeBeforePreparing() throws Exception {
    UserPushToken token = seed(1).getFirst();
    assertBlockedPreparationAfterTokenChange(
````

## 원본 367–477행


````java

  /** Receipt의 토큰 무효화와 신규 묶음 선점이 겹쳐도 무효 토큰을 다시 선점하지 않는다. */
  @Test
  void waitsForInvalidTokenReceiptBeforePreparingNewEvent() throws Exception {
    UserPushToken token = seed(1).getFirst();
    PreparedPushDelivery original =
        deliveries.prepareAll(List.of(command(token, "old"))).getFirst();
    deliveries.recordTicketResults(
        List.of(original.pushDeliveryId()), List.of(PushTicketResult.accepted("ticket")));
    assertBlockedPreparationAfterTokenChange(
        token,
        () ->
            deliveries.recordReceiptResult(
                original.pushDeliveryId(), PushReceiptResult.failed("BadDeviceToken")));
    assertThat(repository.count()).isEqualTo(1);
  }

  private void assertBlockedPreparationAfterTokenChange(UserPushToken token, Runnable change)
      throws Exception {
    CountDownLatch locked = new CountDownLatch(1);
    CountDownLatch commit = new CountDownLatch(1);
    CountDownLatch started = new CountDownLatch(1);
    try (var executor = Executors.newFixedThreadPool(2)) {
      var first =
          executor.submit(
              () ->
                  new TransactionTemplate(transactionManager)
                      .executeWithoutResult(
                          status -> {
                            change.run();
                            locked.countDown();
                            await(commit);
                          }));
      try {
        assertThat(locked.await(5, TimeUnit.SECONDS)).isTrue();
        var second =
            executor.submit(
                () -> {
                  started.countDown();
                  return deliveries.prepareAll(List.of(command(token, "new")));
                });
        assertThat(started.await(5, TimeUnit.SECONDS)).isTrue();
        assertThatThrownBy(() -> second.get(200, TimeUnit.MILLISECONDS))
            .isInstanceOf(TimeoutException.class);
        commit.countDown();
        first.get(5, TimeUnit.SECONDS);
        assertThat(second.get(5, TimeUnit.SECONDS)).isEmpty();
      } finally {
        commit.countDown();
      }
    }
  }

  /** 빈 입력은 SQL 없이 반환하고 제한 초과 입력은 DB에 접근하기 전에 거부한다. */
  @Test
  void validatesBatchBoundaries() {
    final UserPushToken token = seed(1).getFirst();
    dataSource.reset();
    assertThat(deliveries.prepareAll(List.of())).isEmpty();
    deliveries.recordTicketResults(List.of(), List.of());
    assertThatThrownBy(
            () ->
                deliveries.prepareAll(java.util.Collections.nCopies(101, command(token, "event"))))
        .isInstanceOf(IllegalArgumentException.class);
    assertThat(dataSource.sql).isEmpty();
  }

  private List<UserPushToken> seed(int size) {
    return tokens.saveAllAndFlush(
        LongStream.rangeClosed(1, size)
            .mapToObj(
                id ->
                    UserPushToken.register(
                        id, AppPlatform.IOS, "REDACTED_TEST_PUSH_TOKEN"))
            .toList());
  }

  private PreparePushDeliveryCommand command(UserPushToken token, String event) {
    return commandWithOwner(token, event, token.getUserProfileId());
  }

  private PreparePushDeliveryCommand commandWithOwner(
      UserPushToken token, String event, Long owner) {
    return new PreparePushDeliveryCommand(
        owner,
        token.getId(),
        NotificationType.CONTINUE_EXPRESSION,
        NotificationContentVariant.EXPRESSION_DYNAMIC,
        "push:" + event + ":" + owner + ":" + token.getId(),
        "표현 알림",
        "표현을 배워보세요.",
        "/expressions");
  }

  private SendPushNotificationCommand sendCommand(UserPushToken token, String event) {
    return new SendPushNotificationCommand(
        event + ":" + token.getUserProfileId(),
        token.getUserProfileId(),
        NotificationType.CONTINUE_EXPRESSION,
        "표현 알림",
        "표현을 배워보세요.",
        "/expressions");
  }

  private NotificationSender sender() {
    NotificationSender sender = mock(NotificationSender.class);
    when(sender.send(anyList()))
        .thenAnswer(
            call ->
                call.<List<PushMessage>>getArgument(0).stream()
                    .map(m -> PushTicketResult.accepted("ticket-" + m.expoPushToken()))
````


공개본에서는 테스트 대역의 가짜 인증값과 푸시 토큰 리터럴도 `REDACTED_TEST_VALUE` 또는 `REDACTED_TEST_PUSH_TOKEN`으로 대체했다. 실행용 원본은 아니다.
