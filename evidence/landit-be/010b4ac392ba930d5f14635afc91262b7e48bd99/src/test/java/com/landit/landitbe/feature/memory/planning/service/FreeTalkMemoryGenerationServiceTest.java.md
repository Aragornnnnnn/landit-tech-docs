# FreeTalkMemoryGenerationServiceTest.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `010b4ac392ba930d5f14635afc91262b7e48bd99`.
- 원본 경로: `src/test/java/com/landit/landitbe/feature/memory/planning/service/FreeTalkMemoryGenerationServiceTest.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 279–393행

<a id="source-L287"></a>
<a id="source-L328"></a>

````java
  @DisplayName("AI 응답에 신뢰도가 없으면 기억 생성을 실패로 처리한다.")
  @Test
  void failsClosedWhenConfidenceIsMissingFromMappedResponse() throws Exception {
    assertMissingCandidateFieldFails("confidence");
  }

  @DisplayName("기억 저장 대상이 변경되어 있으면 재시도 없이 실패로 처리한다.")
  @Test
  void failsClosedAfterStaleWriteWithoutRetrying() {
    AiMemoryCandidatesResult.Candidate candidate = candidate(0, USER_MESSAGE_ID, "updated fact");
    ConversationMemoryMatch comparable = comparable(909L);
    when(aiClient.extractMemoryCandidates(any()))
        .thenReturn(new AiMemoryCandidatesResult("extractor-v1", List.of(candidate)));
    when(searchRepository.searchActiveComparable(
            any(), eq(USER_PROFILE_ID), eq("chloe"), eq(ConversationMemoryType.EVENT), eq(3)))
        .thenReturn(List.of(comparable));
    when(aiClient.resolveMemory(any()))
        .thenReturn(
            new AiMemoryResolutionResult(
                List.of(
                    new AiMemoryResolutionResult.Resolution(
                        0, AiMemoryOperation.SUPERSEDE, List.of(909L)))));
    when(contextService.persistAndComplete(any(), any()))
        .thenReturn(ConversationMemoryWriteService.PersistenceResult.STALE);

    generationService.generate(LEARNING_SESSION_ID);

    verify(searchRepository)
        .searchActiveComparable(
            any(), eq(USER_PROFILE_ID), eq("chloe"), eq(ConversationMemoryType.EVENT), eq(3));
    verify(aiClient).resolveMemory(any());
    verify(contextService)
        .persistAndComplete(any(ConversationMemoryGenerationRequest.class), any());
    verify(contextService).fail(LEARNING_SESSION_ID);
  }

  @DisplayName("다른 Worker가 선점한 기억 생성 작업은 실행하지 않는다.")
  @Test
  void doesNotRunWhenAnotherWorkerAlreadyClaimedTheJob() {
    when(contextService.claim(LEARNING_SESSION_ID)).thenReturn(null);

    generationService.generate(LEARNING_SESSION_ID);

    verify(aiClient, never()).extractMemoryCandidates(any());
    verify(contextService, never()).persistAndComplete(any(), any());
  }

  @Test
  @DisplayName("AI 시간 초과나 잘못된 응답은 빈 후보 성공으로 처리하거나 저장하지 않는다.")
  void doesNotPersistOrCompleteOnAiFailure() {
    for (ErrorCode code : List.of(ErrorCode.AI_GENERATION_FAILED, ErrorCode.AI_RESPONSE_INVALID)) {
      setUp();
      when(aiClient.extractMemoryCandidates(any())).thenThrow(new ApiException(code));
      generationService.generate(LEARNING_SESSION_ID);
      verify(contextService, never()).persistAndComplete(any(), any());
      verify(aiClient, never()).resolveMemory(any());
      verify(contextService).fail(LEARNING_SESSION_ID);
    }
  }

  private ConversationMemoryGenerationRequest context() {
    return new ConversationMemoryGenerationRequest(
        LEARNING_SESSION_ID,
        USER_PROFILE_ID,
        "chloe",
        "EN",
        "KR",
        "Asia/Seoul",
        List.of(
            new ConversationMemoryHistoryMessage(
                USER_MESSAGE_ID, 1, "USER", "hello", null, USER_OCCURRED_AT),
            new ConversationMemoryHistoryMessage(
                AI_MESSAGE_ID, 1, "AI", "hi", null, USER_OCCURRED_AT.plusMinutes(1))));
  }

  private AiMemoryCandidatesResult.Candidate candidate(
      int index, long sourceMessageId, String content) {
    return new AiMemoryCandidatesResult.Candidate(
        index,
        ConversationMemoryType.EVENT,
        content,
        "KR",
        List.of(sourceMessageId),
        0.8,
        CANDIDATE_VALID_FROM,
        null,
        "openai/text-embedding-3-small",
        embedding());
  }

  private ConversationMemoryMatch comparable(long memoryId) {
    LocalDateTime now = LocalDateTime.of(2026, 8, 24, 10, 0);
    return new ConversationMemoryMatch(
        memoryId, ConversationMemoryType.EVENT, "old fact", now, null, now, 0.1);
  }

  private void assertMissingCandidateFieldFails(String fieldName) throws Exception {
    when(aiClient.extractMemoryCandidates(any())).thenReturn(mappedCandidateWithout(fieldName));
    when(searchRepository.searchActiveComparable(
            any(), eq(USER_PROFILE_ID), eq("chloe"), eq(ConversationMemoryType.EVENT), eq(3)))
        .thenReturn(List.of());

    generationService.generate(LEARNING_SESSION_ID);

    verify(contextService, never()).persistAndComplete(any(), any());
    verify(contextService).fail(LEARNING_SESSION_ID);
  }

  private AiMemoryCandidatesResult mappedCandidateWithout(String fieldName) throws Exception {
    AiMemoryCandidatesResult valid =
        new AiMemoryCandidatesResult(
            "extractor-v1", List.of(candidate(0, USER_MESSAGE_ID, "user fact")));
    JsonNode root = jsonMapper.valueToTree(valid);
    ((ObjectNode) root.get("candidates").get(0)).remove(fieldName);
    return jsonMapper.treeToValue(root, AiMemoryCandidatesResult.class);
````
