# ResponseDemand.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `cfb15ec066277ed0baa255395c02c6c0500e99d7`.
- 원본 경로: `src/main/java/com/landit/landitbe/feature/content/scenario/question/domain/ResponseDemand.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 1–23행


````java
// 시나리오 질문이 요구하는 답변 표현량을 정의한다.

package com.landit.landitbe.feature.content.scenario.question.domain;

import java.math.BigDecimal;

/** 질문별 수준 평가에 적용할 응답 요구도와 가중치다. */
public enum ResponseDemand {
  LOW("0.35"),
  MEDIUM("0.70"),
  HIGH("1.00");

  private final BigDecimal weight;

  ResponseDemand(String weight) {
    this.weight = new BigDecimal(weight);
  }

  /** 질문별 평가에 반영할 가중치를 반환한다. */
  public BigDecimal weight() {
    return weight;
  }
}
````
