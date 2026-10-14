# RewardInboxService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `33ee6ceed257d9233f569f11184d0acaee3ec362`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/reward/service/RewardInboxService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 12–30행

<a id="source-L20-L30"></a>

````java

/** 수신 고유 키 충돌은 호출자가 이 트랜잭션 밖에서 재조회한다. */
@Service
@RequiredArgsConstructor
public class RewardInboxService {
  private final RewardRepository repository;
  private final Clock clock;

  /**
   * 검증된 최소 결제 사실을 영속 수신한다.
   *
   * @param fact 원본 결제 사실
   * @throws org.springframework.dao.DuplicateKeyException 동일 사실이 동시에 수신됐을 때
   */
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void receive(RewardBillingFact fact) {
    if (repository.billing(fact).isEmpty()) {
      repository.insertBilling(fact, clock.instant());
    }
````
