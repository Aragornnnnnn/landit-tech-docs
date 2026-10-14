# test_conversation_api.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `24b56cb3da1e6bd2e0a42bcc67759ea1ad9f84da`.
- 원본 경로: `tests/test_conversation_api.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 4309–4414행

<a id="source-L4317-L4414"></a>

````python
                json=payload,
            )

        self.assertEqual(response.status_code, 200)
        self.assertIsNotNone(response.json()["data"]["levelAssessment"])
        messages = fake_openai.completions.kwargs["messages"]
        self.assertIn("Fabricated fluent answer.", messages[1]["content"])

    def test_assessment_excludes_non_latin_evidence_without_losing_english_turns(self):
        for retry in (False, True):
            with self.subTest(retry=retry):
                payload = valid_session_feedback_payload()
                payload["assessmentMessages"] = valid_assessment_messages()
                payload["assessmentMessages"][0]["userMessage"] = "매워서 피자를 좋아해요."
                assessment = valid_level_assessment()
                for domain in assessment["core"]["messages"][0]["domains"].values():
                    domain["evidenceExcerpt"] = "매워서 피자를 좋아해요."
                response_data = {"sessionId": 100, "levelAssessment": assessment}
                contents = (["{}"] if retry else []) + [json.dumps(response_data)]
                fake = FakeOpenAI(contents=contents)
                with patch("app.core.openai_client.OpenAI", return_value=fake):
                    response = make_client(self._app()).post(
                        "/api/v1/conversation/session-level-assessment", json=payload,
                    )
                self.assertEqual(response.status_code, 200)
                actual = response.json()["data"]["levelAssessment"]
                self.assertIsNotNone(actual)
                for domain in actual["core"]["messages"][0]["domains"].values():
                    self.assertEqual(domain, {
                        "level": None, "evidenceStatus": "NOT_OBSERVED",
                        "evidenceExcerpt": None,
                    })
                self.assertEqual(actual["core"]["messages"][1], assessment["core"]["messages"][1])
                self.assertIsNone(actual["details"])
                self.assertEqual(len(fake.completions.calls), 2 if retry else 1)

    def test_assessment_preserves_names_numeric_choices_and_latin_evidence(self):
        for text in ("My name is 민수.", "Saturday.", "3", "10:30", "Café, please.", "Ｙｅｓ."):
            with self.subTest(text=text):
                payload = valid_session_feedback_payload()
                payload["assessmentMessages"] = valid_assessment_messages()
                payload["assessmentMessages"][0]["userMessage"] = text
                assessment = valid_level_assessment()
                for domain in assessment["core"]["messages"][0]["domains"].values():
                    domain["evidenceExcerpt"] = text
                request = conversation_models.SessionLevelAssessmentRequest.model_validate(payload)
                actual = next_message_service._recover_session_level_assessment(
                    {"sessionId": 100, "levelAssessment": assessment}, request, None,
                )
                self.assertEqual(actual.model_dump(mode="json"), assessment)

    def test_assessment_excludes_only_non_latin_excerpt_domains(self):
        payload = valid_session_feedback_payload()
        payload["assessmentMessages"] = valid_assessment_messages()
        text = "I like 피자 because it is spicy."
        payload["assessmentMessages"][0]["userMessage"] = text
        assessment = valid_level_assessment()
        domains = assessment["core"]["messages"][0]["domains"]
        for domain in domains.values():
            domain["evidenceExcerpt"] = text
        domains["vocabulary"]["evidenceExcerpt"] = "피자"
        request = conversation_models.SessionLevelAssessmentRequest.model_validate(payload)
        actual = next_message_service._recover_session_level_assessment(
            {"sessionId": 100, "levelAssessment": assessment}, request, None,
        )
        self.assertIsNone(actual.core.messages[0].domains.vocabulary.level)
        self.assertEqual(actual.core.messages[0].domains.grammar.level, 4)
        self.assertIsNone(actual.details)
        self.assertEqual(domains["vocabulary"]["level"], 3)

    def test_assessment_excludes_non_latin_scripts_but_keeps_source_validation(self):
        for text in ("저는 개발자예요.", "ㅈㅓㄴㅡㄴ ㅎㅏㄱㅅㅐㅇ", "我喜欢旅行。", "Я люблю путешествия."):
            payload = valid_session_feedback_payload()
            payload["assessmentMessages"] = valid_assessment_messages()
            payload["assessmentMessages"][0]["userMessage"] = text
            assessment = valid_level_assessment()
            for domain in assessment["core"]["messages"][0]["domains"].values():
                domain["evidenceExcerpt"] = text
            request = conversation_models.SessionLevelAssessmentRequest.model_validate(payload)
            data = {"sessionId": 100, "levelAssessment": assessment}
            actual = next_message_service._recover_session_level_assessment(data, request, None)
            self.assertIsNone(actual.core.messages[0].domains.grammar.level)
            assessment["core"]["messages"][0]["domains"]["grammar"]["evidenceExcerpt"] = "없는 근거"
            self.assertIsNone(next_message_service._recover_session_level_assessment(data, request, None))

    def test_assessment_all_non_latin_turns_return_unobserved_core_without_retry(self):
        payload = valid_session_feedback_payload()
        payload["assessmentMessages"] = valid_assessment_messages()
        assessment = valid_level_assessment()
        for message, judgment in zip(payload["assessmentMessages"], assessment["core"]["messages"]):
            message["userMessage"] = "최근 3일은 집에서 쉬었어요."
            for domain in judgment["domains"].values():
                domain["evidenceExcerpt"] = "3"
        fake = FakeOpenAI(content=json.dumps({"sessionId": 100, "levelAssessment": assessment}))
        with patch("app.core.openai_client.OpenAI", return_value=fake):
            response = make_client(self._app()).post(
                "/api/v1/conversation/session-level-assessment", json=payload,
            )
        self.assertEqual(response.status_code, 200)
        actual = response.json()["data"]["levelAssessment"]
        self.assertIsNotNone(actual)
        self.assertIsNone(actual["details"])
        self.assertTrue(all(
            domain["level"] is None and domain["evidenceStatus"] == "NOT_OBSERVED"
            for message in actual["core"]["messages"] for domain in message["domains"].values()
        ))
````
