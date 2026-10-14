# RewardQueryService.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `de231083e2a2afdb01bcb5bb72f50aa11001c9c4`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/reward/service/RewardQueryService.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 364–381행

<a id="source-L372-L381"></a>

````java
          weights += activity.completed() ? 0 : smallTalkOnly ? 30 : splitInTwo ? 15 : 10;
        }
      }
    }
    return (cycle.carriedNumerator() + weights * cycle.maximumWon())
        / cycle.calculationDenominator();
  }

  private String availability(long userId) {
    if (!properties.enabled()) {
      return "DISABLED";
    }
    accounts.lock(userId);
    if (!properties.configured()) {
      return "CONFIGURATION_INCOMPLETE";
    }
    billing.replayUser(userId);
    return accounts.lock(userId).eligibility();
````
