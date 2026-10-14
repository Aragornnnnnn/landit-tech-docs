# request_budget.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `ba5ae416828782b7bf3ff522d522adabaeed2715`.
- 원본 경로: `app/core/request_budget.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 20–76행

<a id="source-L28"></a>
<a id="source-L55"></a>
<a id="source-L65"></a>

````python
    def remaining(self):
        remaining = self.deadline - monotonic()
        if remaining <= 0:
            raise TimeoutError("memory_deadline_exceeded")
        return remaining


@contextmanager
def request_budget(seconds, *, per_call_seconds=20, max_calls=9):
    token = _current.set(RequestBudget(monotonic() + seconds, per_call_seconds, max_calls))
    try:
        yield
    finally:
        _current.reset(token)


def budget_client_options(timeout):
    budget = _current.get()
    if budget is None:
        return {"timeout": timeout, "max_retries": 0} if timeout is not None else {}
    limit = min(budget.remaining(), budget.per_call_seconds,
                timeout if timeout is not None else budget.per_call_seconds)
    return {"timeout": limit, "max_retries": 0,
            "http_client": httpx.Client(transport=_DeadlineTransport(budget, limit))}


def has_request_budget():
    return _current.get() is not None


class _DeadlineTransport(httpx.BaseTransport):
    def __init__(self, budget, call_limit):
        self.budget = budget
        self.call_limit = call_limit

    def handle_request(self, request):
        limit = min(self.budget.remaining(), self.call_limit)
        if self.budget.calls >= self.budget.max_calls:
            raise TimeoutError("memory_call_limit")
        self.budget.calls += 1
        try:
            return asyncio.run(self._send(request, limit))
        except TimeoutError as exc:
            raise httpx.ReadTimeout("memory_deadline_exceeded", request=request) from exc

    async def _send(self, request, limit):
        # read timeout은 청크마다 초기화되므로 전체 송수신에는 취소 가능한 별도 deadline을 둔다.
        async with asyncio.timeout(limit), httpx.AsyncClient(timeout=limit) as client:
            response = await client.request(
                request.method, request.url, headers=request.headers, content=request.read(),
            )
            headers = dict(response.headers)
            # AsyncClient가 이미 압축을 해제했으므로 바깥 Client의 중복 해제를 막는다.
            headers.pop("content-encoding", None)
            headers.pop("content-length", None)
            return httpx.Response(response.status_code, headers=headers,
                                  content=response.content, request=request)
````
