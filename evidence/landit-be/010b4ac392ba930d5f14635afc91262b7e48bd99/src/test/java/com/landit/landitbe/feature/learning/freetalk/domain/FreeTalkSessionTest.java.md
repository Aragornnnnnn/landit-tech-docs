# FreeTalkSessionTest.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `010b4ac392ba930d5f14635afc91262b7e48bd99`.
- 원본 경로: `src/test/java/com/landit/landitbe/feature/learning/freetalk/domain/FreeTalkSessionTest.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 118–189행

<a id="source-L126"></a>

````java

    assertThat(session.getMemoryGenerationStatus()).isEqualTo(MemoryGenerationStatus.READY);
    assertThat(session.getMemoryGenerationStartedAt()).isNull();
  }

  /** 시작된 기억 생성 작업을 실패 처리하면 준비 시각을 지우고 실패 상태로 전환한다. */
  @DisplayName("시작된 기억 생성 작업을 실패 처리하면 준비 시각을 지우고 실패 상태로 전환한다.")
  @Test
  void failsStartedMemoryGeneration() {
    FreeTalkSession session = newSession();
    session.completeByTimeLimit();
    session.prepareMemoryGeneration();
    session.startMemoryGeneration(LocalDateTime.now());

    session.failMemoryGeneration();

    assertThat(session.getConversationStatus()).isEqualTo(FreeTalkConversationStatus.COMPLETED);
    assertThat(session.getMemoryGenerationStatus()).isEqualTo(MemoryGenerationStatus.FAILED);
    assertThat(session.getMemoryGenerationStartedAt()).isNull();
  }

  /** 아직 선점되지 않은 준비 작업도 실패 상태로 정리할 수 있다. */
  @DisplayName("아직 선점되지 않은 준비 작업도 실패 상태로 정리할 수 있다.")
  @Test
  void failsPreparedMemoryGenerationBeforeStart() {
    FreeTalkSession session = newSession();
    session.completeByTimeLimit();
    session.prepareMemoryGeneration();

    session.failMemoryGeneration();

    assertThat(session.getConversationStatus()).isEqualTo(FreeTalkConversationStatus.COMPLETED);
    assertThat(session.getMemoryGenerationStatus()).isEqualTo(MemoryGenerationStatus.FAILED);
    assertThat(session.getMemoryGenerationStartedAt()).isNull();
  }

  /** 완료된 세션은 더 이상 대화 상태를 변경할 수 없다. */
  @DisplayName("완료된 세션은 더 이상 대화 상태를 변경할 수 없다.")
  @Test
  void rejectsConversationStateChangesAfterCompletion() {
    FreeTalkSession session = newSession();
    session.completeByTimeLimit();

    assertThatIllegalStateException().isThrownBy(session::awaitExitDecision);
  }

  /** 사용자 발화 시간은 0 이상인 값만 누적한다. */
  @DisplayName("사용자 발화 시간은 0 이상인 값만 누적한다.")
  @Test
  void accumulatesOnlyNonNegativeSpeakingDuration() {
    FreeTalkSession session = newSession();

    session.addSpeakingDuration(4_200);

    assertThat(session.getAccumulatedSpeakingDurationMs()).isEqualTo(4_200);
    assertThatIllegalStateException().isThrownBy(() -> session.addSpeakingDuration(-1));
  }

  /** 완료된 프리톡 세션에는 사용자 발화 시간을 더할 수 없다. */
  @DisplayName("완료된 프리톡 세션에는 사용자 발화 시간을 더할 수 없다.")
  @Test
  void rejectsSpeakingDurationAfterCompletion() {
    FreeTalkSession session = newSession();
    session.completeByTimeLimit();

    assertThatIllegalStateException().isThrownBy(() -> session.addSpeakingDuration(1));
  }

  private FreeTalkSession newSession() {
    return FreeTalkSession.start(10L, 20L, FreeTalkStartMode.AI_FIRST);
  }
}
````
