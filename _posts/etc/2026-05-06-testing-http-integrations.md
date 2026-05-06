---
layout: post
title: "HTTP-интеграции в тестах: MockWebServer и WireMock"
date: 2026-05-06
categories: [testing, java]
tags: [wiremock, mockwebserver, http, testing, integration]
---

При тестировании сервиса, который выполняет HTTP-запросы во внешние системы, возникает базовая проблема: как проверить поведение без зависимости от реального внешнего API.

Прямое обращение к внешнему сервису в тестах делает их нестабильными, так как результат начинает зависеть от сети и доступности внешней системы. Поведение становится непредсказуемым, время выполнения увеличивается, а ошибки сложно воспроизвести повторно.

По этой причине в тестах внешний сервис не используется напрямую — вместо него поднимается имитация, полностью контролируемая тестом.

### Общая идея

Вместо реального API поднимается локальный HTTP-сервер, который возвращает заранее заданные ответы.

```text
Service → HTTP → Fake Server
```

Такой сервер полностью контролируется тестом: можно задавать статус-коды, тело ответа, задержки и ошибки.

Существует два распространенных инструмента:

- **MockWebServer**
- **WireMock**

### MockWebServer: сценарный подход

MockWebServer — это HTTP-сервер, который возвращает заранее подготовленные ответы в заданной последовательности, это полностью управляемая HTTP-среда для теста.

Такой подход удобен для проверки поведения HTTP-клиента: обработки ошибок, таймаутов, повторных попыток (retry), а также порядка запросов. При этом он не предназначен для тестирования бизнес-логики сервиса — его задача ограничивается моделированием низкоуровневого взаимодействия по HTTP.

Ключевая идея:

```text
Клиент направляется на mock-сервер → все HTTP-вызовы уходят в него
```

#### Полный пример

В данном тесте:

- Поднимается локальный HTTP-сервер;
- Клиент явно направляется на него через `baseUrl`;
- Сервер последовательно возвращает подготовленные ответы;
- Проверяется, как клиент ведет себя при ошибках, задержках и retry.

Рассмотрим пример.

```java
MockWebServer server = new MockWebServer();
server.start();

// 1. Подготавливается поведение сервера (очередь ответов)
server.enqueue(new MockResponse().setResponseCode(503)); // ошибка
server.enqueue(new MockResponse() // таймаут (задержка)
        .setBody("{\"message\":\"retry\"}")
        .setBodyDelay(1500, TimeUnit.MILLISECONDS));
server.enqueue(new MockResponse() // успешный ответ
        .setBody("{\"message\":\"success\"}"));

// 2. Клиент направляется на mock-сервер
WebClient client = WebClient.builder()
        .baseUrl(server.url("/").toString()) // ← ключевой момент
        .build();

// 3. Выполняется HTTP-вызов
Mono<String> response = client.get()
        .uri("/test") // реальный URL будет: http://localhost:<port>/test
        .retrieve()
        .bodyToMono(String.class)
        .retry(2); // клиент сам делает повторные попытки


// 4. Проверка результата
StepVerifier.create(response)
        .expectNext("{\"message\":\"success\"}")
        .verifyComplete();

```

#### Что здесь происходит

```text
1-й запрос → 503 → retry  
2-й запрос → задержка → timeout → retry  
3-й запрос → success  
```

#### Проверка HTTP-вызовов

Дополнительно можно убедиться, что запросы действительно дошли до mock-сервера:

```java
RecordedRequest request1 = server.takeRequest();
RecordedRequest request2 = server.takeRequest();
RecordedRequest request3 = server.takeRequest();

assertThat(request1.getPath()).isEqualTo("/test");
assertThat(request2.getPath()).isEqualTo("/test");
assertThat(request3.getPath()).isEqualTo("/test");
```

#### Область применения

MockWebServer подходит для тестирования:

- retry и timeout логики;
- Последовательности HTTP-вызовов;
- Низкоуровневого поведения клиента.

### WireMock: декларативный подход

WireMock работает по другому принципу. Вместо очереди ответов задаются правила: какой ответ должен быть возвращен при определенном запросе.

Такой подход ориентирован на тестирование бизнес-логики сервиса, который взаимодействует с внешним API. В тесте описывается ожидаемое поведение внешней системы, после чего проверяется, как сервис обрабатывает этот ответ и формирует результат.

#### Пример

В этом примере:

- Поднимается локальный HTTP-сервер WireMock;
- Клиент явно направляется на него через `baseUrl`;
- Задается ожидаемое поведение внешнего API;
- Выполняется вызов и проверяется результат;
- При необходимости проверяется, что запрос действительно был отправлен.

В отличие от MockWebServer, здесь проверяется не последовательность ответов, а корректность обработки конкретного HTTP-контракта.

```java
@RegisterExtension
static WireMockExtension wiremock = WireMockExtension.newInstance()
        .options(wireMockConfig().dynamicPort())
        .build();

// клиент должен быть направлен на WireMock
WebClient webClient = WebClient.builder()
        .baseUrl(wiremock.baseUrl()) // ← ключевой момент
        .build();

// описывается поведение внешнего API
wiremock.stubFor(post("/resolve")
        .willReturn(okJson("""
            {
              "items": [
                { "name": "Pizza", "price": 10.5, "available": true }
              ]
            }
        """)));

// выполняется HTTP-вызов
Mono<Response> response = webClient.post()
        .uri("/resolve")
        .retrieve()
        .bodyToMono(Response.class);

// проверяется результат
StepVerifier.create(response)
        .expectNextMatches(r -> r.getItems().size() == 1)
        .verifyComplete();

// при необходимости проверяется сам факт вызова
wiremock.verify(postRequestedFor(urlEqualTo("/resolve")));
```

#### Область применения

WireMock используется для:

- Интеграционного тестирования сервисов;
- Проверки бизнес-логики;
- Имитации внешних API на уровне контрактов.

### Ключевое различие

```text
MockWebServer → сценарии (очередь ответов)
WireMock      → поведение (правила обработки запросов)
```

MockWebServer удобен для проверки последовательности событий, WireMock — для описания API и работы с ним.

### Независимость от стека

Оба инструмента работают на уровне HTTP и не зависят от используемой технологии (Spring WebFlux, Spring MVC, любой HTTP-клиент). Реактивная модель не является обязательной.

### Что именно тестируется

Важно понимать границы таких тестов.

Проверяется не реальное взаимодействие двух сервисов, а поведение тестируемого сервиса при различных ответах внешней системы.

```text
Service → fake external API
```

Это позволяет детерминировать поведение, воспроизводить ошибки, тестировать крайние сценарии (timeouts, 5xx, частичные ответы).

### Тестирование HTTP-слоя внутри приложения

Отдельный сценарий — тестирование не внешних интеграций, а собственного HTTP API.

В этом случае поднимается само приложение, и запросы отправляются уже в него:

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureWebTestClient
class OrderControllerTest {

    @Autowired
    private WebTestClient webTestClient;
}
```

Здесь поднимается реальный HTTP-сервер приложения (на случайном порту). Данная аннотация запускает полноценный Spring Boot контекст вместе с embedded сервером:

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
```

Предоставляет клиент для отправки HTTP-запросов в этот сервер:

```java
@AutoConfigureWebTestClient
```

#### Пример вызова

```java
webTestClient.post()
        .uri("/api/orders")
        .bodyValue(request)
        .exchange()
        .expectStatus().isCreated();
```

Здесь тестируется HTTP слой самого сервиса (controller → service → repository)

В отличие от предыдущих подходов:

- Здесь нет имитации HTTP-сервера — используется реальный сервер приложения;
- Не проверяется внешний API — тестируется собственный;
- Это не unit-тест, а полноценный integration-тест внутри одного сервиса.

#### Связь с WireMock

Эти подходы часто используются вместе. Входящие запросы тестируются через `WebTestClient`, исходящие — подменяются через WireMock.

```
входящий HTTP → реальный сервер (SpringBootTest) исходящий HTTP → WireMock
```

### Итог

```text
MockWebServer → контроль последовательности HTTP-ответов
WireMock      → имитация поведения внешнего API
SpringBootTest + WebTestClient → тестирование собственного HTTP API
```

Эти подходы решают одну задачу — изолировать тесты от внешних факторов — но применяются на разных уровнях.

MockWebServer используется для проверки поведения HTTP-клиента и сценариев взаимодействия, WireMock — для тестирования логики работы с внешним API, а `SpringBootTest` с `WebTestClient` — для проверки собственного HTTP-интерфейса приложения.

В совокупности они позволяют выстроить предсказуемую и управляемую тестовую среду без зависимости от реальных внешних систем.
