# RewardHistoryService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `33ee6ceed257d9233f569f11184d0acaee3ec362`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/reward/service/RewardHistoryService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 27–47행

<a id="source-L35-L47"></a>

````java
   * 인증된 사용자의 회차 간 내역을 조회한다.
   *
   * @param userId 사용자 ID
   * @param cursor 앞 페이지 커서. 첫 페이지는 null 또는 빈 문자열
   * @param limit 1~100 사이 페이지 크기
   * @return 시각·ID 역순 페이지
   * @throws ApiException 커서 또는 페이지 크기가 유효하지 않을 때
   */
  @Transactional(readOnly = true)
  public RewardHistory.Page page(long userId, String cursor, int limit) {
    if (limit < 1 || limit > 100) {
      throw new ApiException(ErrorCode.INVALID_REQUEST);
    }
    Position position = decode(cursor);
    String availability = rewards.availability(userId);
    if (List.of("DISABLED", "CONFIGURATION_INCOMPLETE").contains(availability)) {
      return new RewardHistory.Page(List.of(), null);
    }
    var rows = repository.page(userId, position, limit + 1);
    var items = List.copyOf(rows.subList(0, Math.min(limit, rows.size())));
    return new RewardHistory.Page(items, rows.size() <= limit ? null : encode(items.getLast()));
````
