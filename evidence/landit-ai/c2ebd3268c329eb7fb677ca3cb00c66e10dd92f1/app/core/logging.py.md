# logging.py — 공개 근거 발췌

- 원본 저장소: `landit-ai` (비공개).
- 원본 커밋: `c2ebd3268c329eb7fb677ca3cb00c66e10dd92f1`.
- 원본 경로: `app/core/logging.py`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–21행

<a id="source-L8-L21"></a>

````python
# AI와 Uvicorn 로그가 실제 로그 레벨을 출력하도록 구성하는 모듈
import logging


LOG_FORMAT = "level=%(levelname)s logger=%(name)s message=%(message)s"


def configure_logging() -> None:
    logging.basicConfig(
        level=logging.WARNING,
        format=LOG_FORMAT,
        force=True,
    )
    for logger_name in ("uvicorn", "uvicorn.error", "uvicorn.access"):
        logger = logging.getLogger(logger_name)
        logger.handlers.clear()
        logger.setLevel(logging.INFO)
        logger.propagate = True

    # 성공한 평가도 호출 메타데이터만 남겨 재시도 전후를 비교할 수 있게 한다.
    logging.getLogger("app.conversation.llm.assessment_observation").setLevel(logging.INFO)
````
