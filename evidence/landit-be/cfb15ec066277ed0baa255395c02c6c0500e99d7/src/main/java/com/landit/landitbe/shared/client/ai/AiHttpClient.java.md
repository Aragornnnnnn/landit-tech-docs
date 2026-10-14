# AiHttpClient.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `cfb15ec066277ed0baa255395c02c6c0500e99d7`.
- 원본 경로: `src/main/java/com/landit/landitbe/shared/client/ai/AiHttpClient.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 63–105행

<a id="source-L71-L105"></a>

````java
   * @param <T> 응답 data 타입
   * @param path API 경로
   * @param payload 요청 값
   * @param responseType 응답 클래스
   * @param requestTimeout 응답 제한 시간
   * @return 변환한 응답 data
   * @throws ApiException 외부 호출 또는 응답 형식이 잘못됐을 때
   */
  public <T> T post(String path, Object payload, Class<T> responseType, Duration requestTimeout) {
    long startNanos = System.nanoTime();
    try {
      HttpRequest request =
          properties
              .authorize(HttpRequest.newBuilder(aiUri(path)))
              .version(HttpClient.Version.HTTP_1_1)
              .header("Accept", "application/json")
              .header("Content-Type", "application/json")
              .timeout(requestTimeout)
              .POST(
                  HttpRequest.BodyPublishers.ofString(
                      jsonMapper.writeValueAsString(payload), StandardCharsets.UTF_8))
              .build();
      HttpResponse<String> response = sendWithinDeadline(request, requestTimeout);
      if (response.statusCode() < 200 || response.statusCode() >= 300) {
        ApiException failure = toApiException(response.statusCode(), response.body());
        if (failure.getStatus().is5xxServerError()) {
          failure.initCause(new AiUpstreamException(response.statusCode()));
        }
        throw failure;
      }
      return readData(response.body(), responseType);
    } catch (ApiException exception) {
      throw exception;
    } catch (InterruptedException exception) {
      Thread.currentThread().interrupt();
      throw ApiException.causedBy(ErrorCode.AI_GENERATION_FAILED, exception);
    } catch (IOException | IllegalArgumentException exception) {
      throw ApiException.causedBy(ErrorCode.AI_GENERATION_FAILED, exception);
    } finally {
      // 성공과 실패를 가리지 않고 왕복 시간을 남겨 지연 구간을 특정한다.
      log.info(
          AI_CALL_ELAPSED_LOG, path, TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startNanos));
    }
````
