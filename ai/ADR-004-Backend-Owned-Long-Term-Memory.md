# ADR 004: Backend가 장기기억 상태와 검색을 소유한다

> 공개 이전: 2026-10-15. 원본은 `landit-ai.wiki`의 `5700ea1bb57a7c6293f99d2a8398735cfb1488be`이다. 본문에 적힌 기준일과 검증 범위를 유지한 스냅샷이며, 현재 배포 상태를 새로 확인한 문서는 아니다. 내부 링크는 공개 근거로 바꾸거나 비공개 참조로 표시했다.


- 상태: `Accepted`
- 결정일: `2026-08-28`

## 맥락

장기기억은 완료 대화에서 후보를 추출하는 LLM 작업뿐 아니라 사용자 권한, 메시지 원문 계보, 중복·대체 상태, pgvector 검색, 동시 쓰기와 사용 trace가 필요합니다. 이 상태를 AI 서버에 두면 Landit Backend의 세션·메시지 원본과 별도 생명주기가 생기고, 재시작·수평 확장·삭제 시 일관성을 다시 해결해야 합니다.

## 결정

Landit Backend가 다음 책임을 소유합니다.

- 세션·메시지 조회와 완료 상태.
- 장기기억 DB schema, 원문 계보와 `ACTIVE`·`SUPERSEDED`·`INVALIDATED` 상태.
- pgvector 검색, 사용자·character 범위와 최대 후보 수.
- 사용자 잠금, 비교 snapshot 재검증과 원자 저장.
- WRITE·USE feature flag, 비동기 작업 상태와 검색·사용 trace.

Landit AI Server는 stateless하게 다음 계산만 수행합니다.

- 완료 세션에서 0~5개 기억 후보와 고정 모델 임베딩 생성.
- Backend가 제공한 비교 기억에 대해 `ADD`, `SUPERSEDE`, `IGNORE` 판정.
- 검색어 임베딩 생성.
- 제공받은 `memoryContext`를 참고한 대화와 `usedMemoryIds` 반환.

## 결과

- 사용자·세션 권한과 삭제 정책을 기존 Backend 데이터 경계 안에서 적용할 수 있습니다.
- AI 서버를 재시작하거나 여러 instance로 늘려도 장기기억 상태가 사라지거나 갈라지지 않습니다.
- AI 판정은 저장 권한이 아닙니다. Backend가 ID 범위와 snapshot을 다시 검증한 뒤에만 적용합니다.
- 기능 연동은 AI·Backend·IaC 세 저장소에 걸치므로 계약과 flag를 함께 검증해야 합니다.
- Backend 장애가 AI 생성 코드와 분리되는 대신, end-to-end 변경은 단일 저장소 테스트만으로 완료를 증명할 수 없습니다.

## 실패 정책

- 쓰기는 부분 저장보다 누락이 안전하므로 fail-closed로 처리합니다. 실패한 생성은 `FAILED`로 기록하되 완료된 사용자 대화는 유지합니다.
- 읽기는 기억 누락보다 대화 중단의 비용이 크므로 fail-open으로 처리합니다. 검색이 실패하면 빈 문맥으로 대화를 계속합니다.
- 사용 trace 저장 실패도 이미 생성된 응답을 취소하지 않습니다.

## 재검토 조건

- Backend가 아닌 별도 데이터 플랫폼이 사용자 기억의 권한·삭제·검색을 공식 소유할 때.
- 현재 pgvector exact 검색이 측정된 규모나 latency 요구를 만족하지 못할 때.
- 매 턴 계획, 민감도 정책 또는 여러 제품이 공유하는 기억처럼 별도 lifecycle이 필요할 때.
- fail-open이 안전하지 않은 기억 유형을 도입할 때.

상세 흐름과 코드 위치는 [Long-Term Memory Architecture](Long-Term-Memory-Architecture.md)를 기준으로 확인합니다.
