# session_assessment_evidence.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `c2ebd3268c329eb7fb677ca3cb00c66e10dd92f1`.
- 원본 경로: `app/conversation/application/session_assessment_evidence.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–50행


````python
# 비라틴 문자만 있는 수준 평가 근거를 제외하고 유효한 영어 관찰값을 보존한다.
import unicodedata

from app.models.conversation import (
    AssessmentEvidenceStatus,
    SessionAssessmentDomain,
    SessionAssessmentMessage,
    SessionLevelAssessmentCore,
    SessionMessageLevelAssessment,
)


def _has_only_non_latin_letters(text: str) -> bool:
    """명백한 비라틴 문자만 제외하며 숫자 응답이나 라틴 문자 언어는 판정하지 않는다."""
    letters = [char for char in unicodedata.normalize("NFKC", text) if char.isalpha()]
    return bool(letters) and not any("LATIN" in unicodedata.name(char, "") for char in letters)


def _filter_message_evidence(
    message: SessionMessageLevelAssessment, user_message: str,
) -> SessionMessageLevelAssessment:
    non_latin_answer = _has_only_non_latin_letters(user_message)
    replacements = {}
    for name in type(message.domains).model_fields:
        domain = getattr(message.domains, name)
        if domain.evidenceStatus == AssessmentEvidenceStatus.OBSERVED and (
            non_latin_answer or _has_only_non_latin_letters(domain.evidenceExcerpt or "")
        ):
            replacements[name] = SessionAssessmentDomain(
                evidenceStatus=AssessmentEvidenceStatus.NOT_OBSERVED,
            )
    if not replacements:
        return message
    return message.model_copy(update={
        "domains": message.domains.model_copy(update=replacements),
    })


def filter_non_latin_assessment_evidence(
    core: SessionLevelAssessmentCore,
    expected_messages: dict[int, SessionAssessmentMessage],
) -> SessionLevelAssessmentCore:
    """ID와 원문 인용 검증 후 호출한다. 변경이 없으면 원본 객체를 반환한다."""
    messages = [
        _filter_message_evidence(message, expected_messages[message.messageId].userMessage)
        for message in core.messages
    ]
    if all(new is old for new, old in zip(messages, core.messages, strict=True)):
        return core
    return core.model_copy(update={"messages": messages})
````
