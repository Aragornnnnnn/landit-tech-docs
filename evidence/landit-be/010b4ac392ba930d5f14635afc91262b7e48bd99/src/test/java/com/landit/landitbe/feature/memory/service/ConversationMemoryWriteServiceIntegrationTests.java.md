# ConversationMemoryWriteServiceIntegrationTests.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `010b4ac392ba930d5f14635afc91262b7e48bd99`.
- 원본 경로: `src/test/java/com/landit/landitbe/feature/memory/service/ConversationMemoryWriteServiceIntegrationTests.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 102–250행

<a id="source-L110"></a>
<a id="source-L145"></a>
<a id="source-L160"></a>
<a id="source-L185"></a>

````java

    assertThat(result).isEqualTo(ConversationMemoryWriteService.PersistenceResult.STORED);
    assertThat(countMemories()).isZero();
    assertThat(memoryGenerationStatus()).isEqualTo("READY");
  }

  @DisplayName("활성 기억 여러 개를 대체하고 새 기억의 출처 계보를 저장한다.")
  @Test
  void supersedesMultipleActiveMemoriesAndStoresTheNewSourceLineage() {
    seedCompletedPreparingSession();
    seedMemory(FIRST_OLD_MEMORY_ID, "first old memory", "ACTIVE");
    seedMemory(SECOND_OLD_MEMORY_ID, "second old memory", "ACTIVE");

    ConversationMemoryWriteService.PersistenceResult result =
        persistAndComplete(
            generationRequest(),
            List.of(
                plan(
                    AiMemoryOperation.SUPERSEDE,
                    List.of(FIRST_OLD_MEMORY_ID, SECOND_OLD_MEMORY_ID),
                    List.of(FIRST_OLD_MEMORY_ID, SECOND_OLD_MEMORY_ID))));

    assertThat(result).isEqualTo(ConversationMemoryWriteService.PersistenceResult.STORED);
    assertThat(countMemories()).isEqualTo(3);
    assertThat(
            jdbcTemplate.queryForList(
                "select status from conversation_memory where id in (?, ?) order by id",
                String.class,
                FIRST_OLD_MEMORY_ID,
                SECOND_OLD_MEMORY_ID))
        .containsExactly("SUPERSEDED", "SUPERSEDED");
    assertThat(
            jdbcTemplate.queryForObject(
                "select count(*) from conversation_memory_source where memory_id not in (?, ?)",
                Integer.class,
                FIRST_OLD_MEMORY_ID,
                SECOND_OLD_MEMORY_ID))
        .isEqualTo(1);
    assertThat(memoryGenerationStatus()).isEqualTo("READY");
  }

  @DisplayName("조회했던 기억 ID와 순서가 바뀌면 오래된 계획으로 판단하고 아무것도 저장하지 않는다.")
  @Test
  void returnsStaleAndWritesNothingWhenOrderedSnapshotIdsChanged() {
    seedCompletedPreparingSession();
    seedMemory(FIRST_OLD_MEMORY_ID, "new comparable memory", "ACTIVE");

    ConversationMemoryWriteService.PersistenceResult result =
        persistAndComplete(
            generationRequest(), List.of(plan(AiMemoryOperation.ADD, List.of(), List.of())));

    assertThat(result).isEqualTo(ConversationMemoryWriteService.PersistenceResult.STALE);
    assertThat(countMemories()).isEqualTo(1);
    assertThat(memoryGenerationStatus()).isEqualTo("PREPARING");
  }

  @DisplayName("대체 대상 하나라도 잘못되면 새 기억과 앞선 대체 작업을 모두 롤백한다.")
  @Test
  void rollsBackNewMemoryAndEarlierSupersedesWhenOneTargetIsInvalid() {
    seedCompletedPreparingSession();
    seedMemory(FIRST_OLD_MEMORY_ID, "first old memory", "ACTIVE");

    assertThatThrownBy(
            () ->
                writeService.persistIfSnapshotCurrent(
                    USER_ID,
                    List.of(
                        plan(
                            AiMemoryOperation.SUPERSEDE,
                            List.of(FIRST_OLD_MEMORY_ID),
                            List.of(FIRST_OLD_MEMORY_ID)),
                        plan(
                            AiMemoryOperation.SUPERSEDE,
                            List.of(FIRST_OLD_MEMORY_ID),
                            List.of(FIRST_OLD_MEMORY_ID)))))
        .isInstanceOf(IllegalStateException.class);

    assertThat(countMemories()).isEqualTo(1);
    assertThat(statusOf(FIRST_OLD_MEMORY_ID)).isEqualTo("ACTIVE");
  }

  @DisplayName("기억 저장 후 세션 완료 처리에 실패하면 기억 저장도 롤백한다.")
  @Test
  void rollsBackMemoryWhenSessionCompletionFailsAfterMemoryWrite() {
    seedCompletedPreparingSession();
    jdbcTemplate.update(
        "update free_talk_session set memory_generation_status = 'READY', "
            + "memory_generation_started_at = null where id = ?",
        FREE_TALK_SESSION_ID);

    ConversationMemoryGenerationRequest request =
        new ConversationMemoryGenerationRequest(
            LEARNING_SESSION_ID, USER_ID, "chloe", "EN", "KR", "Asia/Seoul", List.of());

    assertThatThrownBy(
            () ->
                persistAndComplete(
                    request, List.of(plan(AiMemoryOperation.ADD, List.of(), List.of()))))
        .isInstanceOf(IllegalStateException.class);

    assertThat(countMemories()).isZero();
    assertThat(memoryGenerationStatus()).isEqualTo("READY");
  }

  @DisplayName("스냅샷이나 사용자 범위를 벗어난 대체 대상은 저장 전에 거부한다.")
  @Test
  void rejectsSupersedeTargetOutsideSnapshotAndUserScopeBeforeWriting() {
    seedCompletedPreparingSession();
    seedUser(OTHER_USER_ID);
    seedMemory(OTHER_USER_MEMORY_ID, OTHER_USER_ID, "other user's memory", "ACTIVE");

    assertThatThrownBy(
            () ->
                writeService.persistIfSnapshotCurrent(
                    USER_ID,
                    List.of(
                        plan(
                            AiMemoryOperation.SUPERSEDE,
                            List.of(OTHER_USER_MEMORY_ID),
                            List.of()))))
        .isInstanceOf(IllegalArgumentException.class);

    assertThat(countMemories()).isZero();
    assertThat(statusOf(OTHER_USER_MEMORY_ID)).isEqualTo("ACTIVE");
  }

  @DisplayName("이번 후보를 근거로 한 후속 질문은 방금 저장한 새 기억 ID와 함께 저장한다.")
  @Test
  void storesFollowUpWithTheNewMemoryIdOfItsSourcePlan() {
    seedCompletedPreparingSession();

    persistAndComplete(
        generationRequest(),
        List.of(
            plan(AiMemoryOperation.IGNORE, List.of(), List.of()),
            plan(AiMemoryOperation.ADD, List.of(), List.of())),
        new ConversationMemoryFollowUpDraft(null, 1, "CONCERN", QUESTION, INVITE));

    Long newMemoryId =
        jdbcTemplate.queryForObject(
            "select id from conversation_memory where user_profile_id = ?", Long.class, USER_ID);
    assertThat(followUpRow())
        .containsEntry("USER_PROFILE_ID", USER_ID)
        .containsEntry("MEMORY_ID", newMemoryId)
        .containsEntry("TRIGGER_TYPE", "CONCERN")
        .containsEntry("QUESTION", QUESTION)
        .containsEntry("INVITE", INVITE);
  }

````

## 원본 312–409행

<a id="source-L320"></a>
<a id="source-L344"></a>

````java
    assertThat(followUpRow())
        .containsEntry("MEMORY_ID", FIRST_OLD_MEMORY_ID)
        .containsEntry("TRIGGER_TYPE", "GOAL");
    assertThat(memoryGenerationStatus()).isEqualTo("READY");
  }

  @DisplayName("이번 저장 계획이 질문의 근거 기억을 대체하면 질문만 버리고 기억 저장과 완료는 그대로 한다.")
  @Test
  void skipsFollowUpWhoseSourceMemoryIsSupersededByThisPersistence() {
    seedCompletedPreparingSession();
    seedMemory(FIRST_OLD_MEMORY_ID, "preparing for an interview", "ACTIVE");

    ConversationMemoryWriteService.PersistenceResult result =
        persistAndComplete(
            generationRequest(),
            List.of(
                plan(
                    AiMemoryOperation.SUPERSEDE,
                    List.of(FIRST_OLD_MEMORY_ID),
                    List.of(FIRST_OLD_MEMORY_ID))),
            new ConversationMemoryFollowUpDraft(
                FIRST_OLD_MEMORY_ID, null, "CONCERN", QUESTION, INVITE));

    assertThat(result).isEqualTo(ConversationMemoryWriteService.PersistenceResult.STORED);
    assertThat(statusOf(FIRST_OLD_MEMORY_ID)).isEqualTo("SUPERSEDED");
    assertThat(countFollowUps()).isZero();
    assertThat(countMemories()).isEqualTo(2);
    assertThat(memoryGenerationStatus()).isEqualTo("READY");
  }

  @DisplayName("질문을 만드는 사이 다른 작업이 근거 기억을 이미 대체했으면 질문만 버린다.")
  @Test
  void skipsFollowUpWhoseSourceMemoryWasAlreadySuperseded() {
    seedCompletedPreparingSession();
    seedMemory(FIRST_OLD_MEMORY_ID, "preparing for an interview", "ACTIVE");
    seedMemory(SECOND_OLD_MEMORY_ID, "the interview was cancelled", "ACTIVE");
    jdbcTemplate.update(
        "update conversation_memory set status = 'SUPERSEDED', valid_to = ?, "
            + "superseded_at = ?, superseded_by_id = ? where id = ?",
        NOW,
        NOW,
        SECOND_OLD_MEMORY_ID,
        FIRST_OLD_MEMORY_ID);

    persistAndComplete(
        generationRequest(),
        List.of(),
        new ConversationMemoryFollowUpDraft(
            FIRST_OLD_MEMORY_ID, null, "CONCERN", QUESTION, INVITE));

    assertThat(countFollowUps()).isZero();
    assertThat(memoryGenerationStatus()).isEqualTo("READY");
  }

  @DisplayName("다른 사용자의 기억을 근거로 한 질문은 저장하지 않는다.")
  @Test
  void skipsFollowUpGroundedInAnotherUsersMemory() {
    seedCompletedPreparingSession();
    seedUser(OTHER_USER_ID);
    seedMemory(OTHER_USER_MEMORY_ID, OTHER_USER_ID, "other user's memory", "ACTIVE");

    persistAndComplete(
        generationRequest(),
        List.of(),
        new ConversationMemoryFollowUpDraft(
            OTHER_USER_MEMORY_ID, null, "CONCERN", QUESTION, INVITE));

    assertThat(countFollowUps()).isZero();
    assertThat(memoryGenerationStatus()).isEqualTo("READY");
  }

  @DisplayName("근거 후보가 기억으로 저장되지 않았으면 문구는 남기고 근거 기억만 비운다.")
  @Test
  void storesFollowUpWithoutMemoryWhenItsSourcePlanIsIgnored() {
    seedCompletedPreparingSession();

    persistAndComplete(
        generationRequest(),
        List.of(plan(AiMemoryOperation.IGNORE, List.of(), List.of())),
        new ConversationMemoryFollowUpDraft(null, 0, "HOBBY", QUESTION, INVITE));

    assertThat(followUpRow())
        .containsEntry("MEMORY_ID", null)
        .containsEntry("TRIGGER_TYPE", "HOBBY")
        .containsEntry("QUESTION", QUESTION);
  }

  @DisplayName("근거가 없는 기본 문구도 저장한다.")
  @Test
  void storesDefaultFollowUpWithoutMemory() {
    seedCompletedPreparingSession();

    persistAndComplete(
        generationRequest(),
        List.of(),
        new ConversationMemoryFollowUpDraft(null, null, "NONE", QUESTION, INVITE));

    assertThat(followUpRow())
````
