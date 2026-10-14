# AiHttpClient.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `010b4ac392ba930d5f14635afc91262b7e48bd99`.
- 원본 경로: `src/main/java/com/landit/landitbe/shared/client/ai/AiHttpClient.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 101–174행

<a id="source-L109"></a>

````java
    } finally {
      // 성공과 실패를 가리지 않고 왕복 시간을 남겨 지연 구간을 특정한다.
      log.info(
          AI_CALL_ELAPSED_LOG, path, TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startNanos));
    }
  }

  // HttpRequest.timeout만으로는 헤더 이후 지연된 본문 수신 시간을 제한하지 못한다.
  private HttpResponse<String> sendWithinDeadline(HttpRequest request, Duration timeout)
      throws IOException, InterruptedException {
    var response =
        httpClient.sendAsync(request, HttpResponse.BodyHandlers.ofString(StandardCharsets.UTF_8));
    try {
      return response.get(timeout.toNanos(), TimeUnit.NANOSECONDS);
    } catch (TimeoutException exception) {
      throw new HttpTimeoutException("AI response deadline exceeded");
    } catch (ExecutionException exception) {
      throw new IOException("AI request failed", exception.getCause());
    } finally {
      if (!response.isDone()) {
        response.cancel(true);
      }
    }
  }

  private ApiException toApiException(int statusCode, String responseBody) {
    try {
      JsonNode root = jsonMapper.readTree(responseBody);
      if (statusCode == 400 && root != null) {
        String code = root.path("error").path("code").asString();
        if (ErrorCode.FREE_TALK_CONTEXT_TOO_LARGE.name().equals(code)) {
          return new ApiException(ErrorCode.FREE_TALK_CONTEXT_TOO_LARGE);
        }
        if (ErrorCode.FREE_TALK_SUMMARY_INPUT_TOO_LARGE.name().equals(code)) {
          return new ApiException(ErrorCode.FREE_TALK_SUMMARY_INPUT_TOO_LARGE);
        }
      }
      if (statusCode == 502
          && root != null
          && ErrorCode.AI_RESPONSE_INVALID
              .name()
              .equals(root.path("error").path("code").asString())) {
        return new ApiException(ErrorCode.AI_RESPONSE_INVALID);
      }
    } catch (JacksonException ignored) {
      // 오류 본문을 해석할 수 없으면 외부 AI 호출 실패로 처리한다.
    }
    return new ApiException(ErrorCode.AI_GENERATION_FAILED);
  }

  private <T> T readData(String responseBody, Class<T> responseType) {
    try {
      JsonNode root = jsonMapper.readTree(responseBody);
      JsonNode data = root.get("data");
      if (!root.path("success").asBoolean(false) || data == null || data.isNull()) {
        throw new ApiException(ErrorCode.AI_RESPONSE_INVALID);
      }
      return jsonMapper.treeToValue(data, responseType);
    } catch (ApiException exception) {
      throw exception;
    } catch (JacksonException exception) {
      throw ApiException.causedBy(ErrorCode.AI_RESPONSE_INVALID, exception);
    }
  }

  private URI aiUri(String path) {
    if (properties.baseUrl() == null || properties.baseUrl().isBlank()) {
      throw ApiException.causedBy(
          ErrorCode.AI_GENERATION_FAILED,
          new IllegalStateException("AI base URL is not configured"));
    }
    return URI.create(properties.baseUrl()).resolve(path);
  }
}
````
