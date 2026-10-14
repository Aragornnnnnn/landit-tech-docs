# ConversationMemoryRepository.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `010b4ac392ba930d5f14635afc91262b7e48bd99`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/memory/repository/ConversationMemoryRepository.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 31–104행

<a id="source-L39"></a>

````java
          embedding_model, embedding)
      VALUES (
          :userProfileId, :characterId, :memoryType, :content, :contentLocale,
          :confidence, 'ACTIVE', :validFrom, :validTo, :observedAt, :recordedAt, NULL,
          NULL, NULL, NULL, :extractorVersion, :embeddingModel,
          CAST(:embedding AS extensions.vector))
      """;

  private static final String SUPERSEDE_ACTIVE_SQL =
      """
      UPDATE conversation_memory
      SET status = 'SUPERSEDED', valid_to = :validTo, superseded_at = :supersededAt,
          superseded_by_id = :newMemoryId
      WHERE id = :oldMemoryId
        AND status = 'ACTIVE'
      """;

  private static final String INSERT_SOURCE_SQL =
      """
      INSERT INTO conversation_memory_source (memory_id, session_history_message_id)
      VALUES (:memoryId, :sourceMessageId)
      """;

  private static final String DELETE_RETRIEVAL_TRACE_SQL =
      """
      DELETE FROM free_talk_memory_retrieval
      WHERE free_talk_session_id IN (
          SELECT fts.id
          FROM free_talk_session fts
          JOIN learning_session ls ON ls.id = fts.learning_session_id
          WHERE ls.user_profile_id = :userProfileId)
      """;

  private static final String DELETE_MEMORY_SOURCE_SQL =
      """
      DELETE FROM conversation_memory_source
      WHERE memory_id IN (
          SELECT id FROM conversation_memory WHERE user_profile_id = :userProfileId)
      """;

  private static final String DELETE_MEMORY_SQL =
      "DELETE FROM conversation_memory WHERE user_profile_id = :userProfileId";

  private static final String FIND_RECENT_ACTIVE_SQL =
      """
      SELECT id, memory_type, content, valid_from, valid_to, observed_at
      FROM conversation_memory
      WHERE user_profile_id = :userProfileId
        AND status = 'ACTIVE'
        AND (character_id IS NULL OR character_id = :characterId)
        %s
      ORDER BY observed_at DESC, id DESC
      LIMIT :limit
      """;
  private static final String EXISTS_ACTIVE_SQL =
      """
      SELECT COUNT(*)
      FROM conversation_memory
      WHERE id = :memoryId
        AND user_profile_id = :userProfileId
        AND status = 'ACTIVE'
      """;
  private static final String EXCLUDED_IDS_CONDITION = "AND id NOT IN (:excludedMemoryIds)";

  private final NamedParameterJdbcTemplate jdbcTemplate;

  /**
   * 사용자가 이 캐릭터와 나눌 수 있는 활성 기억을 최근에 말한 순으로 읽는다.
   *
   * <p>범위는 기억 검색과 같다(캐릭터 공용 기억 + 그 캐릭터의 기억). 기억 내용은 컬럼 길이(500자)가 AI 서버의 상한과 같아 따로 거르지 않는다.
   *
   * @param userProfileId 기억 소유 사용자 프로필 ID
   * @param characterId 현재 대화 캐릭터 ID
   * @param excludedMemoryIds 결과에서 뺄 기억 ID. 상한만큼의 자리를 쓸 수 있는 기억으로 채우려고 조회할 때 뺀다
````

## 원본 183–256행

<a id="source-L191"></a>

````java
   *
   * @param memory 저장할 장기기억 입력
   * @param sourceMessageIds 기억의 근거가 되는 원본 메시지 ID 목록
   * @return 생성된 장기기억 ID
   * @throws IllegalArgumentException source ID가 비어 있거나 중복·비양수인 경우
   * @throws IllegalStateException 장기기억 ID를 생성하지 못한 경우
   */
  @Transactional
  public long save(NewConversationMemory memory, List<Long> sourceMessageIds) {
    validateSourceMessageIds(sourceMessageIds);
    KeyHolder keyHolder = new GeneratedKeyHolder();
    jdbcTemplate.update(
        INSERT_MEMORY_SQL, memoryParameters(memory), keyHolder, new String[] {"id"});

    Number memoryId = keyHolder.getKey();
    if (memoryId == null) {
      throw new IllegalStateException("장기기억 ID를 생성하지 못했습니다.");
    }
    jdbcTemplate.batchUpdate(
        INSERT_SOURCE_SQL,
        sourceMessageIds.stream()
            .map(
                sourceMessageId ->
                    new MapSqlParameterSource()
                        .addValue("memoryId", memoryId.longValue())
                        .addValue("sourceMessageId", sourceMessageId))
            .toArray(MapSqlParameterSource[]::new));
    return memoryId.longValue();
  }

  /**
   * 기존 활성 기억을 새 기억으로 조건부 대체하고 유효 기간을 닫는다.
   *
   * @param oldMemoryId 대체할 기존 기억 ID
   * @param newMemoryId 기존 기억이 가리킬 새 기억 ID
   * @param validTo 기존 기억의 유효 종료 시각
   * @param supersededAt 대체 상태가 기록된 시각
   * @return 기존 기억이 활성 상태여서 갱신됐으면 true
   * @throws IllegalArgumentException ID 또는 시각 값이 유효하지 않은 경우
   */
  public boolean supersedeActive(
      long oldMemoryId, long newMemoryId, LocalDateTime validTo, LocalDateTime supersededAt) {
    if (oldMemoryId <= 0 || newMemoryId <= 0 || validTo == null || supersededAt == null) {
      throw new IllegalArgumentException("대체할 장기기억 상태 값이 유효하지 않습니다.");
    }
    int updated =
        jdbcTemplate.update(
            SUPERSEDE_ACTIVE_SQL,
            new MapSqlParameterSource()
                .addValue("oldMemoryId", oldMemoryId)
                .addValue("newMemoryId", newMemoryId)
                .addValue("validTo", validTo)
                .addValue("supersededAt", supersededAt));
    return updated == 1;
  }

  private static MapSqlParameterSource memoryParameters(NewConversationMemory memory) {
    return new MapSqlParameterSource()
        .addValue("userProfileId", memory.userProfileId())
        .addValue("characterId", memory.characterId())
        .addValue("memoryType", memory.memoryType().name())
        .addValue("content", memory.content())
        .addValue("contentLocale", memory.contentLocale().toLanguageTag())
        .addValue("confidence", memory.confidence())
        .addValue("validFrom", memory.validFrom())
        .addValue("validTo", memory.validTo())
        .addValue("observedAt", memory.observedAt())
        .addValue("recordedAt", memory.recordedAt())
        .addValue("extractorVersion", memory.extractorVersion())
        .addValue("embeddingModel", memory.embeddingModel())
        .addValue("embedding", memory.embedding().toString().replace(" ", ""));
  }

  private static void validateSourceMessageIds(List<Long> sourceMessageIds) {
````
