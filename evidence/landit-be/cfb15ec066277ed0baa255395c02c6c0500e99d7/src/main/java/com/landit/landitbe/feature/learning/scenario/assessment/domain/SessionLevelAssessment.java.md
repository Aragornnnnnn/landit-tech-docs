# SessionLevelAssessment.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `cfb15ec066277ed0baa255395c02c6c0500e99d7`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/learning/scenario/assessment/domain/SessionLevelAssessment.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–42행


````java
// 세션별 텍스트 회화 수준 평가 결과를 저장하고 응답한다.

package com.landit.landitbe.feature.learning.scenario.assessment.domain;

import com.fasterxml.jackson.annotation.JsonProperty;
import java.math.BigDecimal;

/** 다섯 영역 점수와 이번 세션의 적용 수준 변경 결과다. */
public record SessionLevelAssessment(
    DomainScore situationPerformance,
    DomainScore grammar,
    DomainScore vocabulary,
    DomainScore discourse,
    DomainScore interactionPragmatics,
    BigDecimal assessedScore,
    Integer assessedLevel,
    boolean sufficientEvidence,
    Source source,
    LearningLevelPolicy.ChangeType changeType,
    Integer previousLevel,
    Integer currentLevel,
    Details details,
    String assessmentVersion) {

  /** 측정값이 없으면 당시 적용 수준 또는 기본값을 결과 화면의 예비 수준으로 제공한다. */
  @JsonProperty("displayLevel")
  public int displayLevel() {
    return assessedLevel != null ? assessedLevel : currentLevel != null ? currentLevel : 3;
  }

  /** 한 평가 영역의 점수와 근거 충족 비율이다. */
  public record DomainScore(BigDecimal score, BigDecimal confidence) {}

  /** 사용자에게 보여줄 선택 설명이다. */
  public record Details(String strength, String improvement) {}

  /** 평가 결과를 만든 출처다. */
  public enum Source {
    MODEL,
    FALLBACK
  }
}
````
