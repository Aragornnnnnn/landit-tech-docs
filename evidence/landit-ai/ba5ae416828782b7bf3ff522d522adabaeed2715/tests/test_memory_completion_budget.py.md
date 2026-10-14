# test_memory_completion_budget.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `ba5ae416828782b7bf3ff522d522adabaeed2715`.
- 원본 경로: `tests/test_memory_completion_budget.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 28–67행

<a id="source-L36"></a>
<a id="source-L44"></a>
<a id="source-L50"></a>

````python
        with patch("app.core.openai_client.OpenAI", return_value=self.client):
            return request_json_completion(
                settings=make_settings(openrouter_api_key="REDACTED_TEST_VALUE", openrouter_model="test"),
                system_prompt="PRIVATE_PROMPT", user_prompt="PRIVATE_USER",
                reasoning_effort="low", response_model=Candidates,
                workflow="free_talk_memory_candidates",
            )

    def test_length_exhaustion_never_retries_even_valid_or_partial_json(self):
        for content in ("", '{"candidates":[', '{"candidates":[]}'):
            with self.subTest(content=content):
                with self.assertRaises(AiResponseInvalidError) as caught:
                    self.invoke([completion(content, "length")])
                self.assertEqual(caught.exception.reason, "completion_token_limit")
                self.assertEqual(self.client.chat.completions.create.call_count, 1)

    def test_blank_is_failure_but_empty_candidates_is_success(self):
        with self.assertRaises(AiResponseInvalidError):
            self.invoke([completion(""), completion("")])
        self.assertEqual(self.client.chat.completions.create.call_count, 2)
        self.assertEqual(self.invoke([completion('{"candidates":[]}')]), {"candidates": []})

    def test_first_failure_and_usage_survive_a_later_provider_failure(self):
        response = completion("", id="gen-123-test", _request_id="req_test",
                              usage=NS(completion_tokens=4096,
                                       completion_tokens_details=NS(reasoning_tokens=4096)))
        with self.assertLogs("app.free_talk.llm", level="INFO") as logs:
            with self.assertRaises(AiGenerationFailedError):
                self.invoke([response, RuntimeError("REDACTED_TEST_VALUE")])
        output = " ".join(logs.output)
        for value in ("completion content is blank", "provider_error", "4096", "gen-123-test",
                      "req_test", "elapsed_ms", "first_failure"):
            self.assertIn(value, output)
        for secret in ("PRIVATE_PROMPT", "PRIVATE_USER", "REDACTED_TEST_VALUE"):
            self.assertNotIn(secret, output)

    def test_missing_diagnostics_and_untrusted_values_are_safe(self):
        with self.assertLogs("app.free_talk.llm", level="INFO") as logs:
            self.invoke([completion('{"candidates":[]}', id="PRIVATE_USER\n", usage=None)])
        self.assertNotIn("PRIVATE_USER", " ".join(logs.output))
````


공개본에서는 테스트 대역의 가짜 인증값과 푸시 토큰 리터럴도 `REDACTED_TEST_VALUE` 또는 `REDACTED_TEST_PUSH_TOKEN`으로 대체했다. 실행용 원본은 아니다.
