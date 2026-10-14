# RewardRepository.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `33ee6ceed257d9233f569f11184d0acaee3ec362`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/reward/repository/RewardRepository.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 200–220행

<a id="source-L208-L220"></a>

````java

  /**
   * 입력 또는 시간 경계가 달라진 회차만 잠근다. 호출자는 계정 잠금을 소유한다.
   *
   * @param userId 사용자 ID
   * @param now 계산 기준 시각
   * @return 정산할 회차
   */
  public List<Cycle> unsettledCycles(long userId, Instant now) {
    return jdbc.query(
        """
        SELECT * FROM reward_cycle WHERE user_id=? AND (
          settled_revision IS NULL OR settled_revision<>input_revision OR settled_at IS NULL
          OR settled_at>? OR settled_until<=?
          OR next_settlement_at IS DISTINCT FROM settled_until)
        ORDER BY id FOR UPDATE
        """,
        this::cycleRow,
        userId,
        time(now),
        time(now));
````

## 원본 637–660행

<a id="source-L645-L660"></a>

````java
  /**
   * 잠근 회차의 입력 버전과 다음 정산 시각을 계산 결과와 같은 트랜잭션에서 확정한다.
   *
   * @param cycle 계산에 사용한 회차와 입력 버전
   * @param now 계산 기준 시각
   * @param next 다음 시간 경계. 더 이상 시간에 의한 변경이 없으면 null
   * @throws IllegalStateException 계산 도중 입력 버전이 변경되었을 때
   */
  public void checkpoint(Cycle cycle, Instant now, Instant next) {
    int updated =
        jdbc.update(
            """
            UPDATE reward_cycle SET settled_revision=?,settled_at=?,settled_until=?,next_settlement_at=?
            WHERE id=? AND input_revision=?
            """,
            cycle.inputRevision(),
            time(now),
            time(next),
            time(next),
            cycle.id(),
            cycle.inputRevision());
    if (updated != 1) {
      throw new IllegalStateException("정산 도중 회차 입력 버전이 변경되었습니다.");
    }
````
