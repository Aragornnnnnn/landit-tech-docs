# RemoteAiMemoryClientTest.java — 공개 근거 발췌

- 원본 저장소: `landit-be` (비공개).
- 원본 커밋: `010b4ac392ba930d5f14635afc91262b7e48bd99`.
- 원본 경로: `src/test/java/com/landit/landitbe/feature/memory/client/ai/RemoteAiMemoryClientTest.java`.
- 공개 이전일: 2026-10-15.
- 아래 행 번호는 원본 기준이다. 생략된 부분은 별도 구간으로 표시하며, 이 문서는 실행 가능한 전체 프로젝트가 아니다.

## 원본 213–344행

<a id="source-L221"></a>
<a id="source-L240"></a>
<a id="source-L265"></a>
<a id="source-L279"></a>

````java
    assertThat(requests.get("/api/v1/free-talk/memory-query-embedding").get("query").asText())
        .isEqualTo(" weekend plans ");
    assertThat(result.embeddingModel()).isEqualTo("openai/text-embedding-3-small");
    assertThat(result.embedding()).hasSize(1536);
  }

  @DisplayName("기억 조회의 성공 응답이 늦으면 응답 전에 시간 초과로 처리한다.")
  @Test
  void memoryQueryTimesOutBeforeDelayedSuccessfulResponse() {
    registerDelayedResponse(
        "/api/v1/free-talk/memory-query-embedding",
        3_000,
        successResponse(
            "{\"embeddingModel\":\"openai/text-embedding-3-small\",\"embedding\":"
                + embeddingJson()
                + "}"));
    long started = System.nanoTime();
    assertGenerationError(
        () ->
            memoryClient(Duration.ofSeconds(5))
                .embedMemoryQuery(new AiMemoryQueryEmbeddingRequest("weekend plans")),
        ErrorCode.AI_GENERATION_FAILED);
    assertThat(Duration.ofNanos(System.nanoTime() - started)).isLessThan(Duration.ofMillis(2_900));
  }

  @Test
  @DisplayName("기억 후보와 판정은 일반 대화와 별개의 55초·25초 제한을 전달한다.")
  void usesDedicatedMemoryTimeouts() {
    try (var construction = mockConstruction(AiHttpClient.class)) {
      RemoteAiMemoryClient client = memoryClient();
      AiMemoryCandidatesRequest candidates = memoryCandidatesRequest();
      AiMemoryResolutionRequest resolution = memoryResolutionRequest();
      client.extractMemoryCandidates(candidates);
      client.resolveMemory(resolution);
      AiHttpClient http = construction.constructed().getFirst();
      verify(http)
          .post(
              "/api/v1/free-talk/memory-candidates",
              candidates,
              AiMemoryCandidatesResult.class,
              Duration.ofSeconds(55));
      verify(http)
          .post(
              "/api/v1/free-talk/memory-resolution",
              resolution,
              AiMemoryResolutionResult.class,
              Duration.ofSeconds(25));
    }
  }

  @Test
  @DisplayName("기억 생성은 짧게 설정된 일반 대화 제한과 독립적으로 응답을 기다린다.")
  void waitsForMemoryIndependentlyOfGeneralTimeout() {
    registerDelayedResponse(
        "/api/v1/free-talk/memory-candidates",
        200,
        successResponse("{\"extractorVersion\":\"v11\",\"candidates\":[]}"));
    assertThat(
            memoryClient(Duration.ofMillis(50))
                .extractMemoryCandidates(memoryCandidatesRequest())
                .candidates())
        .isEmpty();
  }

  @Test
  @DisplayName("HTTP 헤더가 도착해도 응답 본문이 지연되면 전체 제한 시간에 실패한다.")
  void timesOutWhileReceivingBody() {
    server.createContext(
        "/slow-body",
        exchange -> {
          byte[] body = successResponse("{}").getBytes(StandardCharsets.UTF_8);
          exchange.sendResponseHeaders(200, body.length);
          try {
            exchange.getResponseBody().write(body, 0, 1);
            exchange.getResponseBody().flush();
            Thread.sleep(700);
            exchange.getResponseBody().write(body, 1, body.length - 1);
          } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
          } finally {
            exchange.close();
          }
        });
    AiClientProperties properties =
        new AiClientProperties(
            "http://localhost:" + server.getAddress().getPort(),
            "remote",
            "KOREAN_LEARNER",
            Duration.ofSeconds(1),
            Duration.ofSeconds(5),
            Duration.ofSeconds(5),
            Duration.ofSeconds(5));
    long start = System.nanoTime();
    assertGenerationError(
        () ->
            new AiHttpClient(jsonMapper, properties)
                .post("/slow-body", Map.of(), Map.class, Duration.ofMillis(150)),
        ErrorCode.AI_GENERATION_FAILED);
    assertThat(Duration.ofNanos(System.nanoTime() - start)).isLessThan(Duration.ofMillis(600));
  }

  private void registerDelayedResponse(String path, long delayMillis, String response) {
    server.createContext(
        path,
        exchange -> {
          try {
            Thread.sleep(delayMillis);
            byte[] body = response.getBytes(StandardCharsets.UTF_8);
            exchange.sendResponseHeaders(200, body.length);
            exchange.getResponseBody().write(body);
          } catch (InterruptedException exception) {
            Thread.currentThread().interrupt();
          } finally {
            exchange.close();
          }
        });
  }

  private void assertGenerationError(Runnable invocation, ErrorCode expectedErrorCode) {
    assertThatThrownBy(invocation::run)
        .isInstanceOfSatisfying(
            ApiException.class,
            exception -> assertThat(exception.getErrorCode()).isEqualTo(expectedErrorCode));
  }

  private void registerJsonResponse(String path, Map<String, JsonNode> requests, String response)
      throws Exception {
    server.createContext(
        path,
        exchange -> {
          requests.put(
              path,
````
