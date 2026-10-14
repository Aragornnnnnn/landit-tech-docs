# memory_budget.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `ba5ae416828782b7bf3ff522d522adabaeed2715`.
- 원본 경로: `app/free_talk/llm/memory_budget.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–18행

<a id="source-L8"></a>

````python
# 기억 유스케이스의 호출 예산과 생성 실패 계약을 연결한다.
from functools import wraps

from app.core.request_budget import request_budget
from app.free_talk.llm.json_completion import AiGenerationFailedError


def memory_generation_budget(seconds, *, max_calls):
    def decorate(function):
        @wraps(function)
        def bounded(*args, **kwargs):
            try:
                with request_budget(seconds, max_calls=max_calls):
                    return function(*args, **kwargs)
            except TimeoutError as exc:
                raise AiGenerationFailedError("memory_deadline_exceeded") from exc
        return bounded
    return decorate
````
