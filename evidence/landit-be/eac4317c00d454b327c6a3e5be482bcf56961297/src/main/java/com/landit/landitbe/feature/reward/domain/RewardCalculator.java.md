# RewardCalculator.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `eac4317c00d454b327c6a3e5be482bcf56961297`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/reward/domain/RewardCalculator.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 139–157행

<a id="source-L147-L157"></a>

````java
    for (RewardLearningFact fact : facts) {
      int weight = weight(fact, assignment, expressions, conversations);
      long after = Math.addExact(numerator, Math.multiplyExact(input.maximumWon(), weight));
      long earned = after / denominator - numerator / denominator;
      allocations.put(fact.id(), earned);
      balances.put(fact.id(), after / denominator);
      amounts.merge(fact.activityType(), earned, Long::sum);
      numerator = after;
    }
    boolean active = !conversations.isEmpty() || !expressions.isEmpty();
    boolean closed = !input.now().isBefore(until);
    long earned = numerator / denominator - before / denominator;
    long reset = !active && closed ? numerator / denominator : 0;
    long dayMaximum =
        (Math.addExact(before, Math.multiplyExact(30L, input.maximumWon()))) / denominator
            - before / denominator;
    if (!active && closed) {
      numerator = 0;
    }
````
