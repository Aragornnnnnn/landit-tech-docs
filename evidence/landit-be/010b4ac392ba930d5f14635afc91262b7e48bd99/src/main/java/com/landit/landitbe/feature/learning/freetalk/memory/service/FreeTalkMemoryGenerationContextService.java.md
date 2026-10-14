# FreeTalkMemoryGenerationContextService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `010b4ac392ba930d5f14635afc91262b7e48bd99`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/learning/freetalk/memory/service/FreeTalkMemoryGenerationContextService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 141–182행

<a id="source-L149"></a>

````java
   *
   * @param request 장기기억 생성 문맥
   * @param planning 후보별 저장 계획과 후속 질문
   * @return snapshot이 최신이어서 저장과 완료를 수행했으면 STORED, 아니면 STALE
   * @throws ApiException 프리톡 세션을 찾을 수 없을 때
   * @throws IllegalStateException 세션이 실행 중인 장기기억 작업이 아닐 때
   */
  @Transactional
  public ConversationMemoryWriteService.PersistenceResult persistAndComplete(
      ConversationMemoryGenerationRequest request, ConversationMemoryPlanningResult planning) {
    ConversationMemoryPersistence persistence =
        memoryWriteService.persistIfSnapshotCurrent(request.userProfileId(), planning.plans());
    if (persistence.result() == ConversationMemoryWriteService.PersistenceResult.STALE) {
      return persistence.result();
    }
    FreeTalkSession freeTalkSession =
        freeTalkSessionRepository
            .findByLearningSessionIdForUpdate(request.learningSessionId())
            .orElseThrow(() -> new ApiException(SessionErrorCode.SESSION_NOT_FOUND));
    followUpService.record(
        request.userProfileId(),
        freeTalkSession.getId(),
        planning.followUp(),
        persistence.savedMemoryIdsByPlanIndex());
    freeTalkSession.completeMemoryGeneration();
    return persistence.result();
  }

  /**
   * 실행 중인 장기기억 생성 작업을 조건부 실패 상태로 전환한다.
   *
   * @param learningSessionId 실패 처리할 학습 세션 ID
   * @throws IllegalStateException 장기기억 생성 상태가 완료된 프리톡과 일치하지 않을 때
   */
  @Transactional
  public void fail(long learningSessionId) {
    freeTalkSessionRepository
        .findByLearningSessionIdForUpdate(learningSessionId)
        .filter(session -> session.getMemoryGenerationStatus() == MemoryGenerationStatus.PREPARING)
        .ifPresent(FreeTalkSession::failMemoryGeneration);
  }
}
````
