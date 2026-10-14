# TextLevelAssessmentPolicyTest.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `259dfab99d6090c8aa3c94b9c51ddaf081b809a1`.
- 원본 경로: `src/test/java/com/landit/landitbe/feature/learning/scenario/assessment/domain/TextLevelAssessmentPolicyTest.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 84–108행

<a id="source-L92-L108"></a>

````java
                        ResponseDemand.HIGH, null, null, null, null, null)),
                ContentLearningLevel.LEVEL_4_TO_5)
            .orElseThrow();

    assertThat(score.overallConfidence()).isEqualByComparingTo(new BigDecimal("0.50"));
    assertThat(score.sufficientEvidence()).isFalse();
  }

  @DisplayName("전체 관측률뿐 아니라 각 영역에도 충분한 관측 수를 요구한다.")
  @Test
  void requiresEnoughObservationsInEachDomainRatherThanOnlyOverallCoverage() {
    var complete = observation(ResponseDemand.HIGH, 4);
    var partial = new TextLevelAssessmentPolicy.Observation(ResponseDemand.HIGH, 4, 4, 4, 4, null);
    var score =
        TextLevelAssessmentPolicy.calculate(
                List.of(complete, partial), ContentLearningLevel.LEVEL_4_TO_5)
            .orElseThrow();
    assertThat(score.overallConfidence()).isEqualByComparingTo("0.93");
    assertThat(score.sufficientEvidence()).isFalse();
    assertThat(
            TextLevelAssessmentPolicy.calculate(
                    List.of(complete, complete), ContentLearningLevel.LEVEL_4_TO_5)
                .orElseThrow()
                .sufficientEvidence())
        .isTrue();
````
