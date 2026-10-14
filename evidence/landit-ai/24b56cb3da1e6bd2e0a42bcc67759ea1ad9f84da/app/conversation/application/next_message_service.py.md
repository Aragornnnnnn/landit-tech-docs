# next_message_service.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `24b56cb3da1e6bd2e0a42bcc67759ea1ad9f84da`.
- 원본 경로: `app/conversation/application/next_message_service.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1372–1430행

<a id="source-L1380-L1430"></a>

````python
        entry.feedback.messageId: entry.user_message for entry in feedback_entries
    }
    return all(
        cached_user_messages.get(message.messageId) == message.userMessage
        for message in request.assessmentMessages
    )


def _recover_session_level_assessment(
    data: dict[str, Any],
    request: SessionFeedbackRequest | SessionLevelAssessmentRequest,
    feedback_entries: list[_MessageFeedbackCacheEntry] | None,
    require_session_id: bool = True,
) -> SessionLevelAssessment | None:
    if require_session_id and data.get("sessionId") != request.sessionId:
        return None
    raw_assessment = data.get("levelAssessment")
    if not isinstance(raw_assessment, dict) or not request.assessmentMessages:
        return None
    try:
        core = SessionLevelAssessmentCore.model_validate(raw_assessment.get("core"))
    except ValidationError:
        return None
    expected_messages = {
        message.messageId: message for message in request.assessmentMessages
    }
    if feedback_entries is not None and not _assessment_messages_match_cache(
        request,
        feedback_entries,
    ):
        return None
    if [message.messageId for message in core.messages] != request.expectedMessageIds:
        return None
    for message in core.messages:
        expected = expected_messages.get(message.messageId)
        if expected is None:
            return None
        for domain in (
            message.domains.situationPerformance,
            message.domains.grammar,
            message.domains.vocabulary,
            message.domains.discourse,
            message.domains.interactionPragmatics,
        ):
            if (
                domain.evidenceExcerpt is not None
                and domain.evidenceExcerpt not in expected.userMessage
            ):
                return None
    try:
        details = SessionLevelAssessmentDetails.model_validate(
            raw_assessment.get("details"),
        )
    except ValidationError:
        details = None
    filtered_core = filter_non_latin_assessment_evidence(core, expected_messages)
    if filtered_core is not core:
        details = None
    return SessionLevelAssessment(core=filtered_core, details=details)
````
