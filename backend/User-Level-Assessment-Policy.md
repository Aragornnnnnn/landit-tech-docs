# 사용자 수준 평가 정책

> 공개 이전: 2026-10-15. 원본은 `landit-be.wiki`의 `2ee6703b97886858f72bc6ba7c6c7ffe381689ae`이다. 본문에 적힌 기준일과 검증 범위를 유지한 스냅샷이며, 현재 배포 상태를 새로 확인한 문서는 아니다. 내부 링크는 공개 근거로 바꾸거나 비공개 참조로 표시했다.


Landit은 시나리오 답변에서 관찰한 영어 능력을 평가하고, **평가 결과를 보여주는 일과 다음 학습의 난이도를 바꾸는 일을 구분**한다. 높은 점수가 계산돼도 근거가 부족하면 사용자 레벨은 바꾸지 않는다.

이 문서는 계산·반영 정책을 설명한다. HTTP 계약은 API Reference (비공개 내부 문서), 전체 학습 흐름은 Core Product Flows (비공개 내부 문서)를 참고한다.

| 확인 기준 | 값 |
| --- | --- |
| 확인 날짜 | 2026-09-28 |
| BE `main` | `cfb15ec` (비공개 원본 참조) |
| AI `main` | `c2ebd32` (비공개 원본 참조) |
| BE 저장 평가 버전 | `text-level-v1.3` |
| 검증 범위 | 위 커밋의 코드·기존 테스트 대조. 운영 DB·배포·실기기 동작 및 새 LLM 실험은 미검증 |

## 평가 범위와 역할

V1은 **텍스트 회화**만 평가한다. 발음·억양·음성 유창성은 포함하지 않는다. Level 1~5는 영역별 수행을 나타내는 서비스 내부 척도이며 CEFR 등급과 직접 대응하지 않는다.

| 담당 | 판단·처리 |
| --- | --- |
| LLM | 질문별 과제 수행 여부, 영역별 Level 1~5와 원문 근거 판정 |
| AI 서버 | 응답 구조·메시지 ID·원문 인용 검증, 비라틴 문자만 있는 관찰 근거 제외 |
| BE | 저장 원문과 재검증, 가중 점수·근거 충분성 계산, 프로필 반영 여부 결정과 이력 저장 |

서버가 문법·의미의 모든 경우를 직접 판정하지는 않는다. LLM의 의미 판단은 남겨 두되, 계산에 사용할 근거와 사용자 상태 변경 조건을 코드로 제한한다.

## 평가 영역과 Level 1~5

아래는 [현재 AI 루브릭][rubric]의 요약이다. 한 답변도 문법은 Level 3, 어휘는 Level 4처럼 영역별로 다르게 평가할 수 있다.

| 영역·응답 필드 | Level 1 | Level 2 | Level 3 | Level 4 | Level 5 |
| --- | --- | --- | --- | --- | --- |
| 상황 수행 `situationPerformance` | 관련 단어만 제시하며 과제 미완료 | 기본 의도는 전달하나 중요 정보·도움 부족 | 요구된 이유·세부 정보를 포함해 핵심 과제 수행 | 여러 요구를 처리하고 조건에 맞게 대응 | 복잡한 대안을 협상·설득으로 처리 |
| 문법 `grammar` | 문장 구성 없이 단어 나열 | 단순 문장 구사에 제약 | 일반적인 구조를 대체로 정확히 사용 | 다양한 구조를 대체로 정확히 사용 | 복잡한 구조를 유연하고 일관되게 사용 |
| 어휘 `vocabulary` | 기초 단어만 사용하며 표현이 매우 제한됨 | 기초 어휘가 반복되거나 부정확함 | 일상 어휘와 필요한 바꿔 말하기 사용 | 정확하고 자연스러운 어휘·연어 사용 | 뉘앙스와 격식을 유연하게 조절 |
| 대화 구성 `discourse` | 생각이 단절됨 | 생각을 단순 나열 | 앞선 발화 또는 이유·순서·세부 내용을 일관되게 연결 | 이유와 예시로 내용을 명확하게 전개 | 복잡한 내용을 요약·확장하며 구성 |
| 상호작용·화용 `interactionPragmatics` | 응답·도움 요청이 매우 제한됨 | 기초적·직접적이거나 상황에 맞지 않는 응답 | 요청·사과·거절이 적절함 | 상황에 맞게 격식·공손함 조절 | 민감한 협상·의견 차이에 적절하게 대응 |

`taskPerformance`의 `ACHIEVED`는 **질문에서 요구한 일을 수행했다**는 뜻이지 Level 5라는 뜻이 아니다. 짧거나 쉬운 단어로 답했다는 이유만으로 낮은 수준을 부여하지 않으며, 오류 없는 간단한 답변만으로 높은 수준을 입증했다고 보지도 않는다.

## 질문 메타데이터

수준을 평가할 때는 답변뿐 아니라 **질문이 어떤 답변을 요구했는지**도 고려한다. 이름 하나를 묻는 질문과 경험·이유를 설명하도록 요구하는 질문은 관찰 기회가 다르다.

| 정보 | 의미 | 사용 위치 |
| --- | --- | --- |
| 필수 응답 요소 `requiredElements` | 질문에서 명시적으로 답하도록 요구한 내용 | LLM의 과제 수행 판정. 요구하지 않은 설명을 추가 과제로 간주하지 않음 |
| 응답 요구도 `responseDemand` | 답변에 요구되는 설명·구성의 정도 | 영역 점수와 관찰 비율의 질문별 가중치 |
| 질문 그룹 `questionLevelGroup` | 대상 학습자 수준 또는 공통 진단 구분 | 종합 점수의 관찰 상한 |

**응답 요구도와 질문 그룹은 별개다.** 아래 가중치는 사용자가 실제로 길게 답했는지가 아니라 질문에 지정된 요구도에 따른다.

| 응답 요구도 | 질문별 가중치 |
| --- | ---: |
| `LOW` | 0.35 |
| `MEDIUM` | 0.70 |
| `HIGH` | 1.00 |

가중치는 BE의 [`ResponseDemand`][demand]에 정의돼 있다.

| 질문 그룹 | 용도 | 종합 점수 상한 |
| --- | --- | ---: |
| `LEVEL_1` | Level 1용 질문 | 2.00 |
| `LEVEL_2_TO_3` | Level 2~3용 질문 | 4.00 |
| `LEVEL_4_TO_5` | Level 4~5용 질문 | 5.00 |
| `DIAGNOSTIC` | 시나리오 1의 수준 공통 진단 질문 | 5.00 |

관찰 상한은 쉬운 질문의 성공만으로 높은 종합 수준을 부여하지 않기 위한 제한이다. **영역별 점수는 보존하고 종합 점수에만 적용**한다. 따라서 Level 1 질문만으로는 종합 Level 3 이상을 산출하지 못한다. 사용자 능력의 절대 한계가 아니라 해당 질문으로 인정하는 측정 범위다.

질문 메타데이터를 찾을 수 없는 발화에는 그룹별 기본 요구도(`LEVEL_1`: `LOW`, `LEVEL_2_TO_3`·`DIAGNOSTIC`: `MEDIUM`, `LEVEL_4_TO_5`: `HIGH`)를 사용한다. 필수 응답 요소가 없으면 평가 문맥의 내용을 사용한다. [입력 구성 구현][context]

## 관찰값 검증

| 영역별 상태 | 계산에 넣는 값 |
| --- | --- |
| `OBSERVED` | Level 1~5와 해당 사용자 답변에 실제로 포함된 연속 인용문이 필요 |
| `NOT_OBSERVED` | 관찰 기회가 없는 영역. `level`·`evidenceExcerpt`는 `null` |
| `INSUFFICIENT_EVIDENCE` | 기술·처리 문제로 근거가 부족한 영역. `level`·`evidenceExcerpt`는 `null` |

짧은 답변이나 주제 이탈도 판단 가능한 수행이면 관찰값이다. 미관찰을 자동으로 Level 1로 바꾸지 않는다.

AI 서버는 답변 전체와 영역별 인용문에서 글자를 추려, **글자가 있지만 라틴 문자가 하나도 없는 경우** 해당 `OBSERVED` 영역을 `NOT_OBSERVED`로 바꾼다. 근거가 바뀌면 기존 Details도 버린다. 이는 한국어 등 명백한 비라틴 근거의 제외 규칙이지 완전한 영어 언어 감지기는 아니다. 숫자만 있는 답변이나 다른 라틴 문자 언어를 이 규칙으로 판별하지 않는다. [필터 구현][evidence-filter]

BE는 메시지 수·순서·ID, 과제 수행 값, 다섯 영역과 인용문을 다시 검사한다. Core 계약이 깨지면 모델 계산 대신 fallback을 사용한다. 원문 인용 검증은 **없는 문장을 근거로 쓰는 일**을 막으며, 인용이 존재한다고 수준 판정까지 옳아지는 것은 아니다. [검증·저장 구현][assessment-service]

## 점수 계산

영역별로 다음을 계산한다. 여기서 ‘전체 가중치’는 평가 입력에 포함된 모든 답변의 응답 요구도 가중치 합이다.

```text
영역 점수 = Σ(관찰된 Level × 질문 가중치) / Σ(관찰된 질문 가중치)
영역 confidence = Σ(관찰된 질문 가중치) / Σ(전체 질문 가중치)
관찰 횟수 = 해당 영역에 유효한 Level이 있는 서로 다른 답변 수
```

- 미관찰 답변은 점수의 분자·분모에서는 제외하지만, `confidence`의 전체 가중치에는 남긴다.
- 관찰값이 전혀 없으면 해당 영역은 `score=null`, `confidence=0.00`, 관찰 횟수 0이다.
- 영역 점수와 `confidence`는 소수 둘째 자리로 반올림(`HALF_UP`)한다.

| 영역 | 종합 가중치 |
| --- | ---: |
| 상황 수행 | 0.30 |
| 문법 | 0.20 |
| 어휘 | 0.20 |
| 대화 구성 | 0.15 |
| 상호작용·화용 | 0.15 |

```text
assessedScore = min(Σ(영역 점수 × 종합 가중치), 질문 그룹의 상한)
assessedLevel = assessedScore를 정수로 반올림
전체 가중 관찰 비율 = Σ(영역 confidence × 종합 가중치)
```

종합 점수·전체 가중 관찰 비율도 소수 둘째 자리로 반올림한다. 한 영역이라도 점수가 없으면 `assessedScore`와 `assessedLevel`은 `null`이며, 다른 영역의 점수는 보존한다. [계산 구현][score-code]

## 근거 충분성

각 영역이 다음 두 조건을 **모두** 충족해야 한다.

| 조건 | 기준 | 확인하는 것 |
| --- | --- | --- |
| 관찰 횟수 | 서로 다른 답변에서 2회 이상 | 한 답변의 판단에만 의존하지 않는가? |
| 영역 `confidence` | 0.75 이상 | 요구도 가중치를 고려해 충분한 답변 범위를 관찰했는가? |

**다섯 영역 모두 통과해야 `sufficientEvidence=true`다.** 전체 가중 관찰 비율이 높다는 이유로 특정 영역의 부족함을 대체하지 않는다. 이 값은 모델이 스스로 보고한 확신도나 정답 확률이 아니라 **근거의 관찰 범위**다.

### 전체 관찰 비율 93%여도 미반영하는 예

[기존 계산 정책 테스트][coverage-test]의 입력이다. 두 질문 모두 `HIGH`라 가중치는 각각 1이고, 그룹은 `LEVEL_4_TO_5`라 종합 상한은 5다. 관찰된 값에는 모두 Level 4를 부여했다.

| 영역 | 질문 A | 질문 B | 관찰 횟수 | 영역 관찰 비율 | 충분성 |
| --- | ---: | ---: | ---: | ---: | --- |
| 상황 수행 | 4 | 4 | 2 | 1.00 | 충족 |
| 문법 | 4 | 4 | 2 | 1.00 | 충족 |
| 어휘 | 4 | 4 | 2 | 1.00 | 충족 |
| 대화 구성 | 4 | 4 | 2 | 1.00 | 충족 |
| 상호작용·화용 | 4 | 미관찰 | 1 | 0.50 | 부족 |

계산 결과는 `assessedScore=4.00`, `assessedLevel=4`, 전체 가중 관찰 비율 `0.93`이다. 화용 영역은 한 번만 관찰돼 **`sufficientEvidence=false`**다. 이 값을 레벨 반영 정책에 넘기면 `NOT_APPLIED`가 되며 현재 레벨과 승급 신호를 유지한다. 마지막 상태 변경은 계산 결과와 [반영 정책][level-policy]을 연결한 설명이며, 실제 사용자 평가나 새 LLM 실험 결과는 아니다.

일부 근거가 빠졌다고 항상 미반영하는 것도 아니다. 가령 요구도가 `MEDIUM, MEDIUM, HIGH, HIGH`인 네 답변에서 한 영역의 `MEDIUM` 관찰 하나만 빠지면 그 영역은 3회·`2.70/3.40 → 0.79`로 두 조건을 통과한다. 나머지 영역과 프로필 반영 조건도 통과하면 적용 가능하다.

## 사용자 레벨 반영

신규 프로필의 기본 학습 레벨은 3이며, 평가로 수준을 확정했다는 뜻은 아니다.

반영 전에 종합 점수 존재, 전체 가중 관찰 비율 0.75 이상, 다섯 영역의 근거 충분성을 검사한다. 또한 최신 완료 시나리오인지, 평가 요청 뒤 프로필 수준 상태가 갱신되지 않았는지 확인한다. 오래된 비동기 결과가 최신 상태를 덮어쓰지 않도록 하는 조건이다.

| 상태·조건 | 처리 | `changeType` | 승급 신호 |
| --- | --- | --- | --- |
| 최초 유효 평가로 수준 확정 | 종합 점수를 정수 반올림해 적용. 기존 자가 선택값·기본값보다 낮거나 높아도 대체 | `INITIALIZED` | 0으로 초기화 |
| 최초 확정 이후, `assessedScore >= 현재 레벨 + 0.70`을 첫 충족 | 현재 레벨 유지 | `UNCHANGED` | 1회 적립 |
| 위 상향 조건을 유효 평가에서 2회 연속 충족 | 한 단계 승급, 최대 Level 5 | `PROMOTED` | 0으로 초기화 |
| 유효 평가지만 상향 조건 미충족 또는 이미 Level 5 | 현재 레벨 유지. 자동 하향하지 않음 | `UNCHANGED` | 0으로 초기화 |
| 근거 부족·fallback·최신성 조건 미충족 | 수준 변경에 사용하지 않음 | `NOT_APPLIED` | 기존 값 보존 |

여기서 ‘연속’은 달력상 연속 날짜가 아니다. 미적용 결과는 기존 신호를 지우지도 늘리지도 않으며, 적용 가능한 평가가 상향 조건을 충족하지 못하면 신호를 초기화한다. 최초 확정 여부는 구독 도입 이후 평가 이력으로 판단하며, 과거 버전과의 호환 조건도 포함한다. [`LearningLevelPolicy`][level-policy] · [최초 확정 이력 조회][assessment-repository]

`DEMOTED`는 과거 이력 조회 호환을 위해 enum에 남아 있지만 현재 신규 평가에서는 만들지 않는다. 단, **최초 확정 때 기존 설정값보다 낮은 수준으로 바뀌는 경우**는 있을 수 있으며 이때도 `INITIALIZED`다.

## 근거 부족과 fallback의 차이

`source`는 결과 출처이고 `sufficientEvidence`는 레벨 반영에 필요한 근거 충족 여부다. 둘은 별개다.

| 경우 | `source` | 측정값 | 프로필 반영 |
| --- | --- | --- | --- |
| 유효 Core이며 근거도 충분함 | `MODEL` | 영역·종합 점수 | 최신성 조건과 최초 확정·승급 정책에 따름 |
| 유효 Core지만 일부 관찰 부족 | `MODEL` | 계산 가능한 영역·종합 값 보존. 어느 한 영역의 관찰값이 전혀 없으면 종합 값 `null` | `NOT_APPLIED` |
| 유효 Core의 모든 영역이 미관찰 | `MODEL` | 모든 영역 점수와 종합 값 `null`, 관찰 비율 0 | `NOT_APPLIED` |
| 모델 호출 실패·Core 계약 위반·평가 만료 | `FALLBACK` | 모든 영역 점수와 종합 값 `null`, 관찰 비율 0 | `NOT_APPLIED` |

`PREPARING` 상태가 120초 이상 지속되면 조회·저장 경로에서 fallback 종료를 시도한다. fallback 저장에 성공하면 처리 상태는 `COMPLETED`다. **DB 저장 실패 등은 `FAILED`가 될 수 있으므로 모든 장애에서 결과 제공을 보장한다고 표현하지 않는다.** [비동기 생성·복구 구현][generation]

## 응답과 저장

시나리오 완료 후 평가는 최종 피드백과 독립적으로 실행하며, FE는 `GET /api/v1/sessions/{sessionId}/level-assessment`로 상태와 결과를 조회한다. 구독 공개 시각(`LANDIT_SUBSCRIPTION_LAUNCHED_AT`)에 도달해야 활성화되고, 비활성 기간 또는 도입 전 완료 세션은 `data=null`이다. 설정값의 존재만으로 활성화하지 않는다. [조회 API][controller] · [공개 시각 정책][launch-policy]

| 응답 항목 | 의미 |
| --- | --- |
| 다섯 영역의 `score`, `confidence` | 세션에서 계산한 영역별 수준과 가중 관찰 비율 |
| `assessedScore`, `assessedLevel` | 해당 세션의 종합 측정값. 사용자에게 적용된 수준과 다를 수 있음 |
| `sufficientEvidence`, `source` | 근거 충분성·결과 출처. 숫자만으로 유효성을 추정하지 않도록 함께 사용 |
| `previousLevel`, `currentLevel`, `changeType` | 해당 평가 저장 시점의 적용 전후 수준과 처리 결과 |
| `displayLevel` | `assessedLevel` → `currentLevel` → 기본 3 순서의 화면용 값. 평가 성공을 뜻하지 않음 |
| `details.strength`, `details.improvement` | 선택 설명인 잘한 점·개선할 점. 영역별 개별 설명 객체는 아님 |
| `assessmentVersion` | 저장 당시 BE 평가 정책 버전 |

Details는 두 설명이 모두 유효할 때 제공한다. 없거나 잘못돼도 유효한 Core를 버리지 않는다. fallback에는 Details를 붙이지 않는다. 영역별 관찰 횟수와 전체 가중 관찰 비율은 내부 계산에 쓰며, 현재 응답 DTO에는 별도 필드로 노출하지 않는다. [응답 구조][response]

| 저장 위치 | 내용 |
| --- | --- |
| `user_profile.learning_level`, `promotion_streak` | 다음 학습에 적용할 현재 레벨과 연속 승급 신호 |
| `user_level_assessment` | 세션별 다섯 영역 점수·관찰 비율, 종합 값, 충분성, 출처, 전후 수준, 변경 유형, 반영 후 승급 신호, 버전 |
| 같은 테이블의 `core_payload`, `details_payload` | 검증된 질문별 Core와 선택 설명 JSON |

세션 ID에 유일 제약을 두고 잠금·기존 결과 조회 후 저장해 같은 세션의 중복 반영을 방지한다. 과거 응답의 `currentLevel`은 저장 당시 값이며, 조회 순간의 최신 프로필 레벨을 의미하지 않는다. [이력 엔티티][assessment-entity] · [저장 경로][generation]

## 검증 범위와 한계

- [계산 정책 테스트][coverage-test]: 요구도 가중 평균, 종합 점수만의 상한 적용, 단일 답변·미관찰 보존, 영역별 충분성 조건을 검증한다.
- [레벨 반영 테스트][level-test]: 최초 값 대체, 두 번 연속 승급, 근거 부족 시 레벨·신호 보존을 검증한다.
- 가중치·상한·관찰 2회·0.75·승급 차이 0.70은 현재 제품 정책이다. 교육적으로 검증된 절대 기준이나 수준 판정의 정확도 측정 결과로 해석하지 않는다.
- 이 문서는 코드와 기존 테스트의 내용을 대조했다. 이번 문서 갱신을 위해 테스트를 재실행하거나 운영 정확도를 재측정하지는 않았다.



[rubric]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-ai/c2ebd3268c329eb7fb677ca3cb00c66e10dd92f1/app/conversation/application/session_assessment_rubric.py.md
[evidence-filter]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-ai/c2ebd3268c329eb7fb677ca3cb00c66e10dd92f1/app/conversation/application/session_assessment_evidence.py.md
[context]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/learning/scenario/feedback/service/SessionFeedbackContextService.java.md
[demand]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/content/scenario/question/domain/ResponseDemand.java.md
[score-code]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/learning/scenario/assessment/domain/TextLevelAssessmentPolicy.java.md
[coverage-test]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/test/java/com/landit/landitbe/feature/learning/scenario/assessment/domain/TextLevelAssessmentPolicyTest.java.md
[level-policy]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/learning/scenario/assessment/domain/LearningLevelPolicy.java.md
[level-test]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/test/java/com/landit/landitbe/feature/learning/scenario/assessment/domain/LearningLevelPolicyTest.java.md
[assessment-service]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/learning/scenario/assessment/service/SessionLevelAssessmentService.java.md
[assessment-repository]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/learning/scenario/assessment/repository/UserLevelAssessmentRepository.java.md
[assessment-entity]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/learning/scenario/assessment/domain/UserLevelAssessment.java.md
[generation]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/learning/scenario/assessment/service/SessionLevelAssessmentGenerationService.java.md
[response]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/learning/scenario/assessment/domain/SessionLevelAssessment.java.md
[controller]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/learning/scenario/SessionController.java.md
[launch-policy]: https://github.com/Aragornnnnnn/landit-tech-docs/blob/main/evidence/landit-be/cfb15ec066277ed0baa255395c02c6c0500e99d7/src/main/java/com/landit/landitbe/feature/subscription/service/SubscriptionLaunchPolicyService.java.md
