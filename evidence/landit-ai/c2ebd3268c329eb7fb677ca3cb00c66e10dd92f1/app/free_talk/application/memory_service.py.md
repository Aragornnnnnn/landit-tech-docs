# memory_service.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `c2ebd3268c329eb7fb677ca3cb00c66e10dd92f1`.
- 원본 경로: `app/free_talk/application/memory_service.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 260–470행


````python
        "event, so return zero candidates. (4) 'I will visit Osaka from October 3 to October "
        "7, 2026' uses the utterance time as validFrom and "
        "validTo=2026-10-07T23:59:59+09:00. (5) When baseLocale is KR, 'I work at Landit as "
        "a backend engineer, and I play tennis every Wednesday' produces separate PROFILE "
        "contents '사용자는 Landit에서 백엔드 엔지니어로 일한다' and '사용자는 매주 "
        "수요일에 테니스를 친다'. (6) When baseLocale is KR, the ordinary term 'job "
        "interview' is written as '면접', not left in English. (7) When baseLocale is "
        "KR, 'I have a golden retriever named Bori, and we go hiking together every Sunday' "
        "produces separate PROFILE contents '사용자는 보리라는 골든 리트리버를 키운다' "
        "and '사용자는 보리와 매주 일요일에 등산한다'. (8) When baseLocale is KR, 'I "
        "hate cilantro, so I always ask restaurants to leave it out' produces only the "
        "PROFILE content '사용자는 고수를 싫어한다'. (9) occurredAt=2026-09-02T17:00:00+09:00 "
        "and 'Today I started a new job as an engineer at Acme' produces the PROFILE content "
        "'사용자는 Acme에서 엔지니어로 일한다' with the utterance time as validFrom. "
        "(10) 'I visited a gallery with my cousin on September 4, 2026' is an EVENT "
        "with that cousin, never an EPISODE with the listener. (11) 'Maybe I am sick, "
        "but I am not sure. I live alone' keeps only the confirmed PROFILE about living "
        "alone. (12) 'You and I created my practice plan here in this chat' can be an "
        "EPISODE; saving that plan does not create a separate PROFILE. "
        "(13) 'I feel like have the asthma. Not know for sure' produces zero candidates, "
        "including no PROFILE about feeling asthmatic. (14) 'I read Starfield. Its new "
        "edition came out yesterday' does not date the reading: omit that reading EVENT "
        "and do not use the publication date as its validFrom or content date."
    ),
)


class _MemorySupersedeEvidence(BaseModel):
    model_config = ConfigDict(extra="forbid")

    sourceMessageId: int = Field(gt=0)
    quote: str = Field(min_length=1, max_length=500)
    reason: Literal["EXPLICIT_CORRECTION"]

    @field_validator("quote", mode="before")
    @classmethod
    def quote_must_be_trimmed(cls, value: object) -> object:
        return value.strip() if isinstance(value, str) else value

    @field_validator("quote")
    @classmethod
    def quote_must_not_be_blank(cls, value: str) -> str:
        if not value:
            raise ValueError("quote must not be blank")
        return value


class _MemoryResolutionWithEvidence(MemoryResolution):
    supersedeEvidence: _MemorySupersedeEvidence | None = None


class _MemoryResolutionResponseWithEvidence(BaseModel):
    model_config = ConfigDict(extra="forbid")

    resolutions: list[_MemoryResolutionWithEvidence] = Field(min_length=1, max_length=5)


@memory_generation_budget(50, max_calls=9)
def generate_memory_candidates(
    payload: MemoryCandidatesRequest,
    settings: Settings,
) -> MemoryCandidatesResponse:
    """프리톡에서 저장할 기억 후보를 추출한다.

    후보별 임베딩을 생성한다.

    Args:
        payload: 후보 추출에 필요한 세션, 언어, 시간대 및 대화 이력.
        settings: OpenRouter와 임베딩 호출에 사용하는 서버 설정.
    Returns:
        검증된 후보와 서버가 생성한 임베딩을 포함한 응답.
    Raises:
        AiResponseInvalidError: AI 또는 임베딩 응답이 계약을 위반할 때.
        AiGenerationFailedError: AI/임베딩 호출 또는 모델 설정이 실패할 때.
    """
    drafts = _extract_memory_candidate_drafts(payload, settings)
    candidates = _candidates_with_embeddings(drafts, settings) if drafts else []
    return MemoryCandidatesResponse(
        extractorVersion=EXTRACTOR_VERSION,
        candidates=candidates,
        followUpQuestion=generate_follow_up_question(
            payload, candidates, settings, datetime.now(UTC),
        ),
    )


def _extract_memory_candidate_drafts(
    payload: MemoryCandidatesRequest, settings: Settings,
) -> list[MemoryCandidate]:
    """추출과 원문 대조를 마친 후보를 반환하며 임베딩 I/O는 호출자가 담당한다."""
    diagnostics = CompletionAttempts("free_talk_memory_candidates")
    try:
        drafts = _candidate_drafts_with_correction(payload, settings, diagnostics)
    finally:
        diagnostics.log()
    drafts = _filtered_candidate_drafts(
        review_memory_candidates(drafts, _candidate_user_prompt(payload), settings), payload,
    )
    return drafts


def _candidate_drafts_with_correction(
    payload: MemoryCandidatesRequest, settings: Settings, diagnostics: CompletionAttempts,
) -> list[MemoryCandidate]:
    try:
        return _request_candidate_drafts(payload, settings, diagnostics)
    except AiResponseInvalidError as exc:
        if exc.reason != _INVALID_CANDIDATE_SOURCE:
            raise
        source_failure = exc
    try:
        drafts = _request_candidate_drafts(payload, settings, diagnostics, correction=True)
    except Exception as exc:
        exc._memory_source_correction_failed = True
        raise
    observe(workflow="free_talk_memory_candidates", failure_stage="source_validation",
            reason="candidate_source_corrected", outcome="recovered", exc=source_failure, attempt=2)
    return drafts


def _request_candidate_drafts(
    payload: MemoryCandidatesRequest, settings: Settings,
    diagnostics: CompletionAttempts, correction: bool = False,
) -> list[MemoryCandidate]:
    data = request_json_completion(
        settings=settings,
        system_prompt=_candidate_system_prompt() + (" " + _SOURCE_CORRECTION if correction else ""),
        user_prompt=_candidate_user_prompt(payload), reasoning_effort="low",
        response_model=_MemoryCandidateDraftResponse,
        schema_name="free_talk_memory_candidates", workflow="free_talk_memory_candidates",
        max_attempts=1 if correction else 2, diagnostics=diagnostics,
    )
    try:
        return _validated_candidate_drafts(data, payload)
    except AiResponseInvalidError as exc:
        diagnostics.failure(exc.reason)
        raise


def _candidates_with_embeddings(
    drafts: list[MemoryCandidate],
    settings: Settings,
) -> list[MemoryCandidate]:
    """검증된 후보 내용에 서버가 생성한 임베딩을 결합한다."""
    contents = [draft.content.strip() for draft in drafts]
    embeddings = request_embeddings(settings=settings, texts=contents)
    return [
        draft.model_copy(update={"embedding": embedding})
        for draft, embedding in zip(drafts, embeddings, strict=True)
    ]


class _DuplicateMemorySupersedeError(AiResponseInvalidError):
    """여러 후보가 같은 기존 기억을 대체하도록 생성됐을 때 발생한다."""


@memory_generation_budget(20, max_calls=7)
def generate_memory_resolution(
    payload: MemoryResolutionRequest,
    settings: Settings,
) -> MemoryResolutionResponse:
    """장기기억 후보별로 추가, 대체 또는 무시 상태를 판정한다.

    Args:
        payload: 후보와 후보별 비교 대상 장기기억 목록.
        settings: OpenRouter 호출에 사용하는 서버 설정.
    Returns:
        후보마다 하나의 검증된 상태 판정을 포함한 응답.
    Raises:
        AiResponseInvalidError: AI 응답이 후보별 참조 계약을 위반할 때.
        AiGenerationFailedError: AI 호출이나 모델 설정이 실패할 때.
    """
    for candidate in payload.candidates:
        _validate_resolution_sources(candidate)
    try:
        return _request_validated_resolution(payload, settings)
    except _DuplicateMemorySupersedeError:
        logger.warning(
            "장기기억 중복 대체 응답을 한 번 교정합니다. "
            "event=memory_resolution_retry workflow=free_talk_memory_resolution "
            "reason=duplicate_supersede provider=%s model=%s",
            settings.llm_provider,
            settings.openrouter_model,
        )
        return _request_validated_resolution(payload, settings, correction=True)


def _request_validated_resolution(
    payload: MemoryResolutionRequest,
    settings: Settings,
    *,
    correction: bool = False,
) -> MemoryResolutionResponse:
    """교정 요청에도 동일한 계약 검증을 적용하고 추가 재시도는 제한한다."""
    system_prompt = _resolution_system_prompt()
    if correction:
        system_prompt += " " + _DUPLICATE_SUPERSEDE_CORRECTION
    return _validated_resolution(
        request_json_completion(
            settings=settings,
            system_prompt=system_prompt,
            user_prompt=_json_prompt(payload),
            response_model=_MemoryResolutionResponseWithEvidence,
            schema_name="free_talk_memory_resolution",
            workflow="free_talk_memory_resolution",
            max_attempts=1 if correction else 2,
        ),
        payload,
    )


````

## 원본 610–750행


````python
    drafts = []
    for candidate in envelope.candidates:
        raw_candidate = candidate.model_dump(mode="json")
        drafts.append(
            MemoryCandidate.model_validate(
                {
                    **raw_candidate,
                    "embeddingModel": EMBEDDING_MODEL,
                    "embedding": [0.0] * EMBEDDING_DIMENSIONS,
                },
            ),
        )
    return drafts


def _validate_contiguous_indexes(drafts: list[MemoryCandidate]) -> None:
    """후보 인덱스는 0부터 빈틈없이 이어져야 resolution 순서를 보장한다."""
    if [draft.candidateIndex for draft in drafts] != list(range(len(drafts))):
        raise AiResponseInvalidError("candidate indexes must be contiguous")


def _validate_candidate_sources(
    drafts: list[MemoryCandidate],
    payload: MemoryCandidatesRequest,
) -> None:
    """후보 언어와 출처는 요청 기준 언어 및 USER 메시지에만 연결한다."""
    messages_by_id = {message.messageId: message for message in payload.conversationHistory}
    for draft in drafts:
        if draft.contentLocale != payload.baseLocale:
            raise AiResponseInvalidError("candidate locale must match base locale")
        if any(
            messages_by_id.get(source_message_id) is None
            or messages_by_id[source_message_id].role != "USER"
            for source_message_id in draft.sourceMessageIds
        ):
            raise AiResponseInvalidError(_INVALID_CANDIDATE_SOURCE)


def _validate_resolutions(
    resolutions: list[MemoryResolution],
    payload: MemoryResolutionRequest,
) -> None:
    """모든 후보의 resolution과 supersede 대상 격리를 한 번에 검증한다."""
    _validate_resolution_indexes(resolutions, payload)
    comparable_ids_by_candidate = _comparable_ids_by_candidate(payload)
    _validate_superseded_ids(resolutions, comparable_ids_by_candidate)


def _validate_resolution_sources(candidate: MemoryCandidateForResolution) -> None:
    """정정 근거가 있으면 후보의 source ID와 원문 USER 목록을 일치시킨다."""
    if not candidate.sourceMessages:
        return
    source_ids = [message.messageId for message in candidate.sourceMessages]
    if source_ids != candidate.sourceMessageIds:
        raise AiResponseInvalidError("resolution source messages must match source IDs")
    if any(message.role != "USER" for message in candidate.sourceMessages):
        raise AiResponseInvalidError("resolution source must be a user message")


def _ignore_information_losing_supersedes(
    response: _MemoryResolutionResponseWithEvidence,
    payload: MemoryResolutionRequest,
) -> _MemoryResolutionResponseWithEvidence:
    """기존 기억의 세부 정보를 제거하는 역방향 대체를 무시한다."""
    candidates_by_index = {
        candidate.candidateIndex: candidate for candidate in payload.candidates
    }
    guarded = []
    for resolution in response.resolutions:
        candidate = candidates_by_index[resolution.candidateIndex]
        memories_by_id = {
            memory.memoryId: memory for memory in candidate.comparableMemories
        }
        loses_information = any(
            _is_strict_content_subset(
                candidate.content,
                memories_by_id[memory_id].content,
            )
            for memory_id in resolution.supersededMemoryIds
        )
        if (
            resolution.operation == MemoryOperation.SUPERSEDE
            and loses_information
            and not (
                len(resolution.supersededMemoryIds) == 1
                and _has_grounded_supersede_evidence(candidate, resolution.supersedeEvidence)
            )
        ):
            guarded.append(
                resolution.model_copy(
                    update={
                        "operation": MemoryOperation.IGNORE,
                        "supersededMemoryIds": [],
                    },
                ),
            )
            continue
        guarded.append(resolution)
    return _MemoryResolutionResponseWithEvidence(resolutions=guarded)


def _has_grounded_supersede_evidence(
    candidate: MemoryCandidateForResolution,
    evidence: _MemorySupersedeEvidence | None,
) -> bool:
    """명시적 정정 예외는 후보에 연결된 원문 인용이 있을 때만 허용한다."""
    if (
        evidence is None
        or evidence.sourceMessageId not in candidate.sourceMessageIds
        or _EXPLICIT_CORRECTION_CUE_PATTERN.search(evidence.quote) is None
    ):
        return False
    return any(
        message.messageId == evidence.sourceMessageId
        and message.role == "USER"
        and evidence.quote in message.content
        and not _LANGUAGE_EXAMPLE_PATTERN.search(message.content)
        and not _NON_FACT_CORRECTION_PATTERN.search(message.content)
        for message in candidate.sourceMessages
    )


def _is_strict_content_subset(candidate: str, existing: str) -> bool:
    """후보가 기존 기억의 단어를 제거한 일반화인지 보수적으로 판정한다."""
    candidate_tokens = _content_tokens(candidate)
    existing_tokens = _content_tokens(existing)
    return candidate_tokens < existing_tokens


def _content_tokens(content: str) -> set[str]:
    tokens = {
        _strip_korean_particle(token.lower())
        for token in _CONTENT_TOKEN_PATTERN.findall(content)
    }
    return tokens - {""}


def _strip_korean_particle(token: str) -> str:
    """한국어 조사 차이로 같은 핵심 단어가 달라지는 것을 방지한다."""
    for suffix in _KOREAN_PARTICLE_SUFFIXES:
        if token.endswith(suffix) and len(token) > len(suffix) + 1:
````
