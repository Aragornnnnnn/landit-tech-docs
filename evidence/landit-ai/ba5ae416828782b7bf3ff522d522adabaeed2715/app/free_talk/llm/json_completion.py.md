# json_completion.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `ba5ae416828782b7bf3ff522d522adabaeed2715`.
- 원본 경로: `app/free_talk/llm/json_completion.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 81–154행

<a id="source-L89"></a>

````python
            request["response_format"] = {"type": "json_object"}
            completion = _create_completion(client, request, attempts)
            return _parse_json_object(_extract_content(completion))

        request["response_format"] = json_schema_response_format(
            response_model,
            name=schema_name,
        )
        for attempt in range(1, max_attempts + 1):
            try:
                completion = _create_completion(client, request, attempts)
            except Exception as exc:
                if not structured_outputs_unsupported(exc):
                    raise
                logger.warning(
                    "Structured Outputs를 지원하지 않아 json_object로 전환합니다. "
                    "event=structured_output_fallback workflow=%s provider=%s model=%s "
                    "fromFormat=json_schema toFormat=json_object "
                    "attempt=%s maxAttempts=%s",
                    workflow,
                    settings.llm_provider,
                    model,
                    attempt,
                    max_attempts,
                )
                request["response_format"] = {"type": "json_object"}
                try:
                    completion = _create_completion(client, request, attempts)
                except Exception as fallback_exc:
                    if not structured_outputs_unsupported(fallback_exc):
                        raise
                    logger.warning(
                        "json_object를 지원하지 않아 기존 프롬프트 방식으로 전환합니다. "
                        "event=structured_output_fallback workflow=%s provider=%s "
                        "model=%s fromFormat=json_object toFormat=prompt "
                        "attempt=%s maxAttempts=%s",
                        workflow,
                        settings.llm_provider,
                        model,
                        attempt,
                        max_attempts,
                    )
                    request.pop("response_format", None)
                    completion = _create_completion(client, request, attempts)
                data = _parse_json_object(_extract_content(completion))
                observe(workflow=workflow, failure_stage="output_format", reason="format_fallback",
                        outcome="recovered", attempt=attempt)
                return data
            try:
                data = _parse_json_object(_extract_content(completion))
            except AiResponseInvalidError as exc:
                attempts.failure(exc.reason)
                logger.warning(
                    "Structured Outputs JSON 형식 검증에 실패했습니다. "
                    "event=json_format_failure workflow=%s provider=%s model=%s "
                    "reason=%s attempt=%s maxAttempts=%s",
                    workflow,
                    settings.llm_provider,
                    model,
                    type(exc).__name__,
                    attempt,
                    max_attempts,
                )
                if attempt == max_attempts:
                    raise
                logger.warning(
                    "Structured Outputs JSON 형식 오류를 재시도합니다. "
                    "event=structured_output_retry workflow=%s provider=%s model=%s "
                    "nextAttempt=%s maxAttempts=%s",
                    workflow,
                    settings.llm_provider,
                    model,
                    attempt + 1,
                    max_attempts,
````

## 원본 199–250행

<a id="source-L207"></a>

````python
        )
        raise AiGenerationFailedError from exc
    finally:
        attempts.log()

    raise AiGenerationFailedError


def _create_completion(client, request, attempts):
    completion = attempts.create(client, request)
    if attempts.exhausted():
        raise AiResponseInvalidError("completion_token_limit")
    return completion


def _resolved_model(settings: Settings, override: str | None) -> str:
    """호출별 모델 지정이 있으면 그것을, 없으면 기본 OPENROUTER_MODEL을 쓴다."""
    if override is not None and override.strip():
        return override.strip()
    return _required_model(settings)


def _required_model(settings: Settings) -> str:
    if settings.openrouter_model is None or not settings.openrouter_model.strip():
        raise AiGenerationFailedError("OPENROUTER_MODEL is required.")
    return settings.openrouter_model


def _extract_content(completion: Any) -> str:
    try:
        content = completion.choices[0].message.content
    except (AttributeError, IndexError) as exc:
        raise AiResponseInvalidError("completion content is missing") from exc
    if not isinstance(content, str) or not content.strip():
        raise AiResponseInvalidError("completion content is blank")
    return content.strip()


def _parse_json_object(content: str) -> dict[str, object]:
    try:
        data = json.loads(content)
    except JSONDecodeError as exc:
        raise AiResponseInvalidError(
            "completion is not valid JSON",
            raw_content=content,
        ) from exc
    if not isinstance(data, dict):
        raise AiResponseInvalidError(
            "completion must be a JSON object",
            raw_content=content,
        )
    return data
````
