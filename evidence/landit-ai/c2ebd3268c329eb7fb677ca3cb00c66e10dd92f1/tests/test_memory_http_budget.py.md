# test_memory_http_budget.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `c2ebd3268c329eb7fb677ca3cb00c66e10dd92f1`.
- 원본 경로: `tests/test_memory_http_budget.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–139행


````python
# 실제 로컬 HTTP 지연으로 기억 호출의 전체 예산과 SDK 재시도 제한을 검증한다.
import gzip
import json
import threading
import time
import unittest
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

from openai import APIConnectionError, APITimeoutError

from app.core.openai_client import create_openai_client
from app.core.request_budget import request_budget
from app.free_talk.application.memory_service import _extract_memory_candidate_drafts
from app.free_talk.llm.json_completion import AiGenerationFailedError
from app.models.free_talk import MemoryCandidatesRequest
from tests.test_free_talk_api import (
    make_settings, valid_memory_candidate_completion, valid_memory_candidates_payload,
)


class MemoryHttpBudgetTests(unittest.TestCase):
    def setUp(self):
        self.delays = []
        self.drip = False
        self.compressed = False
        self.calls = 0
        self.content = '{"candidates":[]}'
        owner = self

        class Handler(BaseHTTPRequestHandler):
            def log_message(self, *args):
                pass

            def do_POST(self):
                index = owner.calls
                owner.calls += 1
                self.rfile.read(int(self.headers["Content-Length"]))
                time.sleep(owner.delays[index])
                body = json.dumps({"id": "gen-local", "choices": [{
                    "message": {"content": owner.content}, "finish_reason": "stop",
                }]}).encode()
                if owner.compressed:
                    body = gzip.compress(body)
                try:
                    self.send_response(200)
                    if owner.compressed:
                        self.send_header("Content-Encoding", "gzip")
                    self.send_header("Content-Length", str(len(body)))
                    self.end_headers()
                    for byte in (bytes([b]) for b in body) if owner.drip else [body]:
                        self.wfile.write(byte)
                        self.wfile.flush()
                        if owner.drip:
                            time.sleep(0.03)
                except (BrokenPipeError, ConnectionResetError):
                    pass

        self.server = ThreadingHTTPServer(("127.0.0.1", 0), Handler)
        self.thread = threading.Thread(target=self.server.serve_forever, daemon=True)
        self.thread.start()
        self.settings = make_settings(openrouter_api_key="REDACTED_TEST_VALUE", openrouter_model="test",
                                      openrouter_base_url=f"http://127.0.0.1:{self.server.server_port}")

    def tearDown(self):
        self.server.shutdown()
        self.server.server_close()
        self.thread.join()

    def call(self):
        with create_openai_client(self.settings) as client:
            return client.chat.completions.create(model="test", messages=[])

    def test_per_call_deadline_disables_sdk_retry(self):
        self.delays = [0.8]
        start = time.monotonic()
        with request_budget(2, per_call_seconds=0.2), self.assertRaises(APITimeoutError):
            self.call()
        self.assertLess(time.monotonic() - start, 0.7)
        self.assertEqual(self.calls, 1)

    def test_successive_calls_share_remaining_budget(self):
        self.delays = [0.12, 0.7]
        start = time.monotonic()
        with request_budget(0.4, per_call_seconds=1), self.assertRaises(APITimeoutError):
            self.call()
            self.call()
        self.assertLess(time.monotonic() - start, 0.7)
        self.assertEqual(self.calls, 2)

    def test_trickling_body_cannot_extend_deadline(self):
        self.delays = [0]
        self.drip = True
        start = time.monotonic()
        with request_budget(0.2), self.assertRaises(APITimeoutError):
            self.call()
        self.assertLess(time.monotonic() - start, 0.7)

    def test_exhausted_budget_starts_no_request_and_scope_is_reset(self):
        with request_budget(0.01):
            time.sleep(0.02)
            with self.assertRaises(TimeoutError):
                create_openai_client(self.settings)
        self.assertEqual(self.calls, 0)
        with create_openai_client(self.settings) as client:
            self.assertEqual(client.max_retries, 2)

    def test_call_limit_prevents_additional_http_attempts(self):
        self.delays = [0]
        with request_budget(2, max_calls=1):
            self.call()
            with self.assertRaises(APIConnectionError):
                self.call()
        self.assertEqual(self.calls, 1)

    def test_compressed_provider_response_is_decoded_once(self):
        self.delays = [0]
        self.compressed = True
        with request_budget(2):
            self.assertEqual(self.call().choices[0].message.content, '{"candidates":[]}')

    def test_source_correction_shares_remaining_time(self):
        self.content = json.dumps(valid_memory_candidate_completion(sourceMessageIds=[3001]))
        self.delays = [0.12, 0.7]
        start = time.monotonic()
        with request_budget(0.4, per_call_seconds=1), self.assertRaises(AiGenerationFailedError):
            _extract_memory_candidate_drafts(
                MemoryCandidatesRequest(**valid_memory_candidates_payload()), self.settings,
            )
        self.assertLess(time.monotonic() - start, 0.7)
        self.assertEqual(self.calls, 2)

    def test_source_correction_cannot_bypass_http_call_limit(self):
        self.content = json.dumps(valid_memory_candidate_completion(sourceMessageIds=[3001]))
        self.delays = [0]
        with request_budget(2, max_calls=1), self.assertRaises(AiGenerationFailedError):
            _extract_memory_candidate_drafts(
                MemoryCandidatesRequest(**valid_memory_candidates_payload()), self.settings,
            )
        self.assertEqual(self.calls, 1)
````


공개본에서는 테스트 대역의 가짜 인증값과 푸시 토큰 리터럴도 `REDACTED_TEST_VALUE` 또는 `REDACTED_TEST_PUSH_TOKEN`으로 대체했다. 실행용 원본은 아니다.
