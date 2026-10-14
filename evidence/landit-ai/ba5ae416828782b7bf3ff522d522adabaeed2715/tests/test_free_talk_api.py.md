# test_free_talk_api.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `ba5ae416828782b7bf3ff522d522adabaeed2715`.
- 원본 경로: `tests/test_free_talk_api.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 946–1068행

<a id="source-L1003"></a>
<a id="source-L954"></a>

````python
                "content": "I practice the cello every Thursday.", "translatedContent": None,
            }]) | {"memoryContext": [memory]},
            FakeOpenAI(contents=[json.dumps(completion)]),
        )

        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()["data"]["usedMemoryIds"], [])

    def test_memory_resolution_requires_grounded_factual_correction_for_subset(self):
        cases = [
            ("I still practice the cello, but no longer every Thursday.", True),
            ("이제 목요일마다 하는 건 아니라 첼로를 연습할 때마다 달라.", True),
            ("I practice the cello.", False),
            ("Actually, I practice the cello.", False),
            ("For English practice: I no longer practice the cello every Thursday.", False),
            ("If I no longer practice the cello every Thursday, what should I do?", False),
        ]
        for source, allowed in cases:
            with self.subTest(source=source):
                payload = valid_memory_resolution_payload()
                candidate = payload["candidates"][0]
                candidate.update(memoryType="PROFILE", content="사용자는 첼로를 연습한다.")
                candidate["comparableMemories"][0]["content"] = "사용자는 매주 목요일에 첼로를 연습한다."
                candidate["sourceMessages"][0]["content"] = source
                resolution = {
                    "candidateIndex": 0, "operation": "SUPERSEDE", "supersededMemoryIds": [77],
                    "supersedeEvidence": {
                        "sourceMessageId": 3002, "quote": source, "reason": "EXPLICIT_CORRECTION",
                    },
                }
                response = self._post(
                    "/api/v1/free-talk/memory-resolution", payload,
                    FakeOpenAI(contents=[json.dumps({"resolutions": [resolution]})]),
                )
                self.assertEqual(response.status_code, 200)
                result = response.json()["data"]["resolutions"][0]
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
        response = self._post(
            "/api/v1/free-talk/turn",
            valid_turn_payload(
                topic={
                    "topicId": None,
                    "title": None,
                    "promptDescription": None,
                },
            ),
            fake_openai,
        )

        self.assertEqual(response.status_code, 200)
        user_prompt = fake_openai.completions.calls[0]["messages"][1]["content"]
        self.assertIsNone(json.loads(user_prompt)["topic"])

    def test_turn_ignores_unsupported_emotion(self):
        response = self._post(
            "/api/v1/free-talk/turn",
            valid_turn_payload(),
            FakeOpenAI(
                contents=[json.dumps(normal_turn_completion(emotion="friendly"))],
            ),
        )

        self.assertEqual(response.status_code, 200)
        self.assertIsNone(response.json()["data"]["emotion"])

````
