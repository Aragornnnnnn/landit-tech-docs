# test_free_talk_api.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `e524025d431ef214a29bd06bf3cd51a8c12c75cd`.
- 원본 경로: `tests/test_free_talk_api.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 917–975행


````python
                self.assertEqual(result["operation"], "SUPERSEDE" if allowed else "IGNORE")
                self.assertNotIn("supersedeEvidence", result)

    def test_memory_resolution_rejects_fabricated_or_missing_correction_evidence(self):
        for evidence in (None, {"sourceMessageId": 3002, "quote": "no longer true",
                                "reason": "EXPLICIT_CORRECTION"}):
            with self.subTest(evidence=evidence):
                payload = valid_memory_resolution_payload()
                candidate = payload["candidates"][0]
                candidate["content"] = "사용자는 첼로를 연습한다."
                candidate["comparableMemories"][0]["content"] = "사용자는 매주 목요일에 첼로를 연습한다."
                response = self._post(
                    "/api/v1/free-talk/memory-resolution", payload,
                    FakeOpenAI(contents=[json.dumps({"resolutions": [{
                        "candidateIndex": 0, "operation": "SUPERSEDE", "supersededMemoryIds": [77],
                        "supersedeEvidence": evidence,
                    }]})]),
                )
                self.assertEqual(response.status_code, 200)
                self.assertEqual(response.json()["data"]["resolutions"][0]["operation"], "IGNORE")

    def test_memory_time_policy_preserves_history_across_local_date_boundaries(self):
        for zone, expected in (("Asia/Seoul", "2026-09-06T00:00:00+09:00"),
                               ("America/Los_Angeles", "2026-09-05T08:00:00-07:00")):
            with self.subTest(zone=zone), patch(
                "app.free_talk.application.conversation_service.datetime",
            ) as clock:
                clock.now.return_value = datetime(2026, 9, 5, 15, tzinfo=UTC)
                memory = valid_memory_context(content="사용자는 2026-09-04에 면접이 예정되어 있었다.")
                memory.update(validFrom="2026-09-01T10:00:00+09:00", validTo=None)
                for path, payload, completion in (
                    ("opening", valid_opening_payload(), opening_completion()),
                    ("turn", valid_turn_payload(), normal_turn_completion()),
                ):
                    fake = FakeOpenAI(contents=[json.dumps(completion)])
                    response = self._post(f"/api/v1/free-talk/{path}",
                                          payload | {"timezone": zone, "memoryContext": [memory]}, fake)
                    self.assertEqual(response.status_code, 200)
                    messages = fake.completions.calls[0]["messages"]
                    policy = messages[0]["content"]
                    self.assertIn(expected, policy)
                    for rule in ("future validFrom", "calendar dates in content", "inclusive",
                                 "not evidence that the event happened", "historical memories"):
                        self.assertIn(rule, policy)
                    self.assertEqual(json.loads(messages[1]["content"])["memoryContext"][0]["content"],
                                     memory["content"])

    def test_turn_normalizes_duplicate_used_memory_ids(self):
        response = self._post(
            "/api/v1/free-talk/turn",
            valid_turn_payload() | {"memoryContext": [valid_memory_context()]},
            FakeOpenAI(contents=[json.dumps(normal_turn_completion(usedMemoryIds=[77, 77]))]),
        )

        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()["data"]["usedMemoryIds"], [])

    def test_turn_treats_all_null_topic_as_absent(self):
        fake_openai = FakeOpenAI(contents=[json.dumps(normal_turn_completion())])
````

## 원본 2313–2371행


````python
        self.assertIn("사용자는 Acme에서 엔지니어로 일한다", system_prompt)
        self.assertIn("Every EPISODE content must explicitly name", system_prompt)

    def test_memory_candidates_rejects_ai_message_as_source(self):
        fake_openai = FakeOpenAI(
            contents=[
                json.dumps(
                    valid_memory_candidate_completion(sourceMessageIds=[3001]),
                ),
            ],
        )

        response = self._post(
            "/api/v1/free-talk/memory-candidates",
            valid_memory_candidates_payload(),
            fake_openai,
        )

        self.assertEqual(response.status_code, 502)
        self.assertEqual(response.json()["error"]["code"], "AI_RESPONSE_INVALID")
        self.assertEqual(len(fake_openai.embeddings.calls), 0)

    def test_memory_candidates_rejects_non_contiguous_candidate_index(self):
        fake_openai = FakeOpenAI(
            contents=[
                json.dumps(
                    {
                        "candidates": [
                            valid_memory_candidate_completion()["candidates"][0]
                            | {"candidateIndex": 1},
                        ],
                    },
                ),
            ],
        )

        response = self._post(
            "/api/v1/free-talk/memory-candidates",
            valid_memory_candidates_payload(),
            fake_openai,
        )

        self.assertEqual(response.status_code, 502)
        self.assertEqual(response.json()["error"]["code"], "AI_RESPONSE_INVALID")

    def test_memory_candidates_allows_empty_candidate_list(self):
        fake_openai = FakeOpenAI(contents=[json.dumps({"candidates": []})])

        response = self._post(
            "/api/v1/free-talk/memory-candidates",
            valid_memory_candidates_payload(),
            fake_openai,
        )

        self.assertEqual(response.status_code, 200)
        self.assertEqual(
            response.json()["data"],
            {
                "candidates": [],
````

## 원본 2940–2998행


````python
        self.assertEqual(response.status_code, 502)
        self.assertEqual(response.json()["error"]["code"], "AI_RESPONSE_INVALID")

    def test_memory_resolution_rejects_unlinked_source_before_llm_call(self):
        payload = valid_memory_resolution_payload()
        payload["candidates"][0]["sourceMessages"][0]["messageId"] = 9999
        fake_openai = FakeOpenAI(contents=[json.dumps({"resolutions": []})])

        response = self._post(
            "/api/v1/free-talk/memory-resolution",
            payload,
            fake_openai,
        )

        self.assertEqual(response.status_code, 502)
        self.assertEqual(response.json()["error"]["code"], "AI_RESPONSE_INVALID")
        self.assertEqual(len(fake_openai.completions.calls), 0)

    def test_memory_resolution_ignores_candidate_that_removes_existing_detail(self):
        payload = valid_memory_resolution_payload()
        payload["candidates"][0]["memoryType"] = "PROFILE"
        payload["candidates"][0]["content"] = (
            "사용자는 Nori와 매주 토요일에 산책한다"
        )
        payload["candidates"][0]["comparableMemories"][0]["content"] = (
            "사용자는 Nori와 매주 토요일에 서울숲에서 산책한다"
        )
        fake_openai = FakeOpenAI(
            contents=[
                json.dumps(
                    {
                        "resolutions": [
                            {
                                "candidateIndex": 0,
                                "operation": "SUPERSEDE",
                                "supersededMemoryIds": [77],
                            },
                        ],
                    },
                ),
            ],
        )

        response = self._post(
            "/api/v1/free-talk/memory-resolution",
            payload,
            fake_openai,
        )

        self.assertEqual(response.status_code, 200)
        self.assertEqual(
            response.json()["data"]["resolutions"][0],
            {
                "candidateIndex": 0,
                "operation": "IGNORE",
                "supersededMemoryIds": [],
            },
        )

````

## 원본 3109–3193행


````python
            "IGNORE",
        )

    def test_memory_resolution_rejects_unknown_superseded_memory(self):
        fake_openai = FakeOpenAI(
            contents=[
                json.dumps(
                    {
                        "resolutions": [
                            {
                                "candidateIndex": 0,
                                "operation": "SUPERSEDE",
                                "supersededMemoryIds": [999],
                            },
                        ],
                    },
                ),
            ],
        )

        response = self._post(
            "/api/v1/free-talk/memory-resolution",
            valid_memory_resolution_payload(),
            fake_openai,
        )

        self.assertEqual(response.status_code, 502)
        self.assertEqual(response.json()["error"]["code"], "AI_RESPONSE_INVALID")

    def test_memory_resolution_rejects_superseded_ids_for_ignore(self):
        fake_openai = FakeOpenAI(
            contents=[
                json.dumps(
                    {
                        "resolutions": [
                            {
                                "candidateIndex": 0,
                                "operation": "IGNORE",
                                "supersededMemoryIds": [77],
                            },
                        ],
                    },
                ),
            ],
        )

        response = self._post(
            "/api/v1/free-talk/memory-resolution",
            valid_memory_resolution_payload(),
            fake_openai,
        )

        self.assertEqual(response.status_code, 502)
        self.assertEqual(response.json()["error"]["code"], "AI_RESPONSE_INVALID")

    def test_memory_resolution_rejects_memory_from_another_candidate(self):
        payload = valid_memory_resolution_payload()
        payload["candidates"].append(
            {
                "candidateIndex": 1,
                "content": "사용자는 새 직장에 적응 중이다.",
                "memoryType": "EVENT",
                "sourceMessageIds": [3002],
                "observedAt": "2026-08-29T19:20:00+09:00",
                "comparableMemories": [
                    {
                        "memoryId": 88,
                        "content": "사용자는 새 직장을 시작했다.",
                        "validFrom": "2026-08-25T20:10:00+09:00",
                        "validTo": None,
                        "observedAt": "2026-08-25T20:10:00+09:00",
                    },
                ],
            },
        )
        fake_openai = FakeOpenAI(
            contents=[
                json.dumps(
                    {
                        "resolutions": [
                            {
                                "candidateIndex": 0,
                                "operation": "SUPERSEDE",
                                "supersededMemoryIds": [88],
                            },
````
