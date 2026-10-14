# test_memory_source_correction.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `c2ebd3268c329eb7fb677ca3cb00c66e10dd92f1`.
- 원본 경로: `tests/test_memory_source_correction.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–152행


````python
# 기억 후보의 잘못된 출처 교정과 실패 진단을 검증한다.
import json
import unittest
from types import SimpleNamespace as NS
from unittest.mock import Mock, patch

from app.core.request_budget import request_budget
from app.free_talk.application.memory_service import _extract_memory_candidate_drafts
from app.free_talk.llm.json_completion import AiGenerationFailedError
from app.main import create_app
from app.models.free_talk import MemoryCandidatesRequest
from tests.test_free_talk_api import (
    FakeOpenAI, make_client, make_settings,
    valid_memory_candidate_completion, valid_memory_candidates_payload,
)
from tests.test_memory_completion_budget import completion


class MemorySourceCorrectionTests(unittest.TestCase):
    def setUp(self):
        self.settings = make_settings(openrouter_api_key="REDACTED_TEST_VALUE", openrouter_model="test")
        self.payload = valid_memory_candidates_payload()
        self.valid = valid_memory_candidate_completion()

    def post(self, contents):
        self.fake = FakeOpenAI(contents=[json.dumps(item) for item in contents])
        with (
            patch("app.core.openai_client.OpenAI", return_value=self.fake),
            patch("app.free_talk.application.memory_service.review_memory_candidates",
                  side_effect=lambda drafts, *_: drafts) as review,
            patch("app.common.failure_observation.sentry_sdk.capture_exception") as capture,
        ):
            response = make_client(create_app(self.settings)).post(
                "/api/v1/free-talk/memory-candidates", json=self.payload,
            )
        self.review_calls = review.call_count
        self.events = capture.call_args_list
        return response

    def test_invalid_sources_are_regenerated_once_and_reviewed(self):
        for ids in ([3001], [9999], [3002, 3001]):
            with self.subTest(ids=ids):
                response = self.post([
                    valid_memory_candidate_completion(sourceMessageIds=ids), self.valid,
                ])
                self.assertEqual(response.status_code, 200)
                self.assertEqual(response.json()["data"]["candidates"][0]["sourceMessageIds"], [3002])
                self.assertEqual(len(self.fake.completions.calls), 2)
                self.assertEqual(self.review_calls, 1)
                self.assertEqual(len(self.fake.embeddings.calls), 1)
                self.assertEqual(self.events, [])
                calls = self.fake.completions.calls
                self.assertEqual(json.loads(calls[0]["messages"][1]["content"])["allowedSourceMessageIds"], [3002])
                self.assertIn("Regenerate", calls[1]["messages"][0]["content"])

    def test_repeated_invalid_source_fails_without_downstream_work(self):
        bad = valid_memory_candidate_completion(sourceMessageIds=[3001])
        response = self.post([bad, bad, self.valid])
        self.assertEqual(response.status_code, 502)
        self.assertEqual(len(self.fake.completions.calls), 2)
        self.assertEqual(self.review_calls, 0)
        self.assertEqual(self.fake.embeddings.calls, [])
        self.assertEqual(self.fake.completions.follow_up_calls, [])
        self.assertEqual(len(self.events), 1)
        tags = self.events[0].kwargs["tags"]
        self.assertEqual(tags["workflow"], "free_talk_memory_candidates")
        self.assertEqual(tags["reason"], "candidate_source_correction_failed")

    def test_correction_does_not_relax_other_contracts(self):
        for corrected in (
            valid_memory_candidate_completion(contentLocale="EN"),
            valid_memory_candidate_completion(candidateIndex=1),
        ):
            with self.subTest(corrected=corrected):
                response = self.post([
                    valid_memory_candidate_completion(sourceMessageIds=[9999]), corrected,
                ])
                self.assertEqual(response.status_code, 502)
                self.assertEqual(self.fake.embeddings.calls, [])
                self.assertEqual(len(self.fake.completions.calls), 2)

    def test_corrected_source_still_passes_through_actual_review(self):
        bad = valid_memory_candidate_completion(sourceMessageIds=[3001])
        review = {"reviews": [{"candidateIndex": 0, "isPersonal": False,
                  "eventDateIsGrounded": False, "isStableProfile": False,
                  "reason": "Synthetic unsupported fact.", "decision": "DROP"}]}
        fake = FakeOpenAI(contents=[json.dumps(item) for item in (bad, self.valid, review)])
        with patch("app.core.openai_client.OpenAI", return_value=fake):
            response = make_client(create_app(self.settings)).post(
                "/api/v1/free-talk/memory-candidates", json=self.payload,
            )
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()["data"]["candidates"], [])
        self.assertEqual(len(fake.completions.calls), 3)
        self.assertEqual(fake.embeddings.calls, [])

    def test_blank_or_length_limited_correction_is_not_retried(self):
        for corrected in (completion(""), completion('{"candidates":[]}', "length")):
            with self.subTest(corrected=corrected):
                client = Mock()
                client.chat.completions.create.side_effect = [
                    completion(json.dumps(valid_memory_candidate_completion(sourceMessageIds=[3001]))),
                    corrected,
                ]
                with patch("app.core.openai_client.OpenAI", return_value=client):
                    response = make_client(create_app(self.settings)).post(
                        "/api/v1/free-talk/memory-candidates", json=self.payload,
                    )
                self.assertEqual(response.status_code, 502)
                self.assertEqual(client.chat.completions.create.call_count, 2)
                client.embeddings.create.assert_not_called()

    def test_correction_may_return_genuine_empty_candidates(self):
        response = self.post([
            valid_memory_candidate_completion(sourceMessageIds=[3001]), {"candidates": []},
        ])
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json()["data"]["candidates"], [])
        self.assertEqual(self.fake.embeddings.calls, [])
        self.assertEqual(len(self.fake.completions.calls), 2)

    def test_exhausted_deadline_does_not_send_correction(self):
        fake = FakeOpenAI(contents=[json.dumps(valid_memory_candidate_completion(sourceMessageIds=[3001]))])
        with (
            request_budget(50),
            patch("app.core.openai_client.OpenAI", return_value=fake),
            patch("app.core.request_budget.monotonic", side_effect=[0, float("inf")]),
            self.assertRaises(AiGenerationFailedError),
        ):
            _extract_memory_candidate_drafts(MemoryCandidatesRequest(**self.payload), self.settings)
        self.assertEqual(len(fake.completions.calls), 1)

    def test_source_failure_preserves_provider_diagnostics_at_warning_level(self):
        bad = json.dumps(valid_memory_candidate_completion(sourceMessageIds=[3001]))
        client = Mock()
        client.chat.completions.create.side_effect = [
            completion(bad, id="gen-first", usage=NS(completion_tokens=42,
                       completion_tokens_details=NS(reasoning_tokens=12))),
            RuntimeError("PRIVATE_PROVIDER_BODY"),
        ]
        with (
            patch("app.core.openai_client.OpenAI", return_value=client),
            self.assertLogs("app.free_talk.llm", level="WARNING") as logs,
            self.assertRaises(AiGenerationFailedError),
        ):
            _extract_memory_candidate_drafts(MemoryCandidatesRequest(**self.payload), self.settings)
        output = " ".join(logs.output)
        for expected in ("candidate source must be a user message", "gen-first", '"completion_tokens": 42',
                         '"reasoning_tokens": 12', '"finish_reason": "stop"', "provider_error", "elapsed_ms"):
            self.assertIn(expected, output)
        for private in ("REDACTED_TEST_VALUE", "PRIVATE_PROVIDER_BODY", "면접", "3001", "interview"):
            self.assertNotIn(private, output)
````


공개본에서는 테스트 대역의 가짜 인증값과 푸시 토큰 리터럴도 `REDACTED_TEST_VALUE` 또는 `REDACTED_TEST_PUSH_TOKEN`으로 대체했다. 실행용 원본은 아니다.
