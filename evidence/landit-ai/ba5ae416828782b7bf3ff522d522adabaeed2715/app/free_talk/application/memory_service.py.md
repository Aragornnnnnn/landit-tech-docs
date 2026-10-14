# memory_service.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `ba5ae416828782b7bf3ff522d522adabaeed2715`.
- 원본 경로: `app/free_talk/application/memory_service.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 367–440행

<a id="source-L375"></a>

````python
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


def _validated_resolution(
    data: dict[str, object],
    payload: MemoryResolutionRequest,
) -> MemoryResolutionResponse:
    """AI resolution 응답을 형식과 후보별 참조 범위까지 검증한다."""
    try:
        response = _MemoryResolutionResponseWithEvidence.model_validate(data)
        _validate_resolutions(response.resolutions, payload)
        guarded = _ignore_information_losing_supersedes(response, payload)
        return MemoryResolutionResponse(
            resolutions=[
                MemoryResolution.model_validate(
````

## 원본 619–734행

<a id="source-L627"></a>
<a id="source-L669"></a>

````python
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
            token = token[: -len(suffix)]
            break
    for suffix in _KOREAN_VERB_SUFFIXES:
        if token == suffix:
            return ""
        if token.endswith(suffix) and len(token) > len(suffix):
            return token[: -len(suffix)]
    return token


def _validate_resolution_indexes(
    resolutions: list[MemoryResolution],
    payload: MemoryResolutionRequest,
) -> None:
    """요청 후보마다 정확히 하나의 고유한 resolution이 있어야 한다."""
    requested_indexes = [candidate.candidateIndex for candidate in payload.candidates]
    resolved_indexes = [resolution.candidateIndex for resolution in resolutions]
    if sorted(requested_indexes) != sorted(resolved_indexes):
        raise AiResponseInvalidError("every candidate must have one resolution")
    if len(resolved_indexes) != len(set(resolved_indexes)):
        raise AiResponseInvalidError("candidate resolutions must be unique")


def _comparable_ids_by_candidate(
    payload: MemoryResolutionRequest,
) -> dict[int, set[int]]:
````
