# DailyScenarioQueryService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `eac4317c00d454b327c6a3e5be482bcf56961297`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/learning/scenario/selection/service/DailyScenarioQueryService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 22–40행

<a id="source-L30-L40"></a>

````java
  /**
   * 당일 배정을 준비하고 날짜별 시나리오 응답을 조회한다.
   *
   * @param userId 사용자 ID
   * @param date 조회 날짜. null이면 KST 오늘
   * @return 시나리오 응답
   * @throws ApiException 미래 날짜이거나 사용자·시나리오 정보를 찾을 수 없을 때
   */
  public DailyScenarioResponse getDailyScenario(long userId, LocalDate date) {
    var evaluatedAt = clock.instant();
    var today = evaluatedAt.atZone(ZoneId.of("Asia/Seoul")).toLocalDate();
    var queryDate = date == null ? today : date;
    if (queryDate.isAfter(today)) {
      throw new ApiException(ErrorCode.INVALID_REQUEST);
    }
    if (queryDate.equals(today)) {
      rewardAssignments.capture(userId, evaluatedAt);
    }
    return scenarios.getDailyScenario(userId, queryDate, evaluatedAt);
````
