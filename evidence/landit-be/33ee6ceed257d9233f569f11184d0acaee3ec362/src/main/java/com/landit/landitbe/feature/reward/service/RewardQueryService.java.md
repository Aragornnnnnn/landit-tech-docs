# RewardQueryService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `33ee6ceed257d9233f569f11184d0acaee3ec362`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/reward/service/RewardQueryService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 28–39행

<a id="source-L36-L39"></a>

````java
  private final ProfileSubscriptionService profiles;

  /**
   * 현재 도전과 이전 만기 회차의 합계를 조회한다.
   *
   * @param userId 인증된 사용자
   * @return 보상 현황
   */
  @Transactional(
      readOnly = true,
      isolation = org.springframework.transaction.annotation.Isolation.REPEATABLE_READ)
  public RewardView summary(long userId) {
````
