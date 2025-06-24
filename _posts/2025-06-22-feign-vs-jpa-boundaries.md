---
title: "Изоляция данных и Feign: архитектура без сквозных связей"
layout: post
date: 2025-06-22 18:00:00 +0300
categories: [microservices]
tags: [feign, spring-cloud, microservices, architecture, domain, spring-boot, inter-service-communication]
---

В микросервисной архитектуре важнейшим принципом является **изоляция данных**. Каждый сервис должен быть независимым и автономным: он владеет своей базой данных, управляет своим доменом и не зависит от внутренних деталей других сервисов — ни на уровне кода, ни на уровне БД.

Но как тогда один сервис может узнать что-то о другом? Ответ — через **внешний API**. И в мире Spring Boot для этого удобно использовать **FeignClient**.

## Почему нельзя связывать микросервисы через JPA

Рассмотрим пример. У нас есть два сервиса:

- `courses-service` — управляет курсами и уроками.
- `content-service` — отвечает за хранение медиафайлов, например видео-лекций.

В `Lesson` из `courses-service` может быть ссылка на контент. Но мы **не должны** делать это так:

```java
@OneToOne
@JoinColumn(name = "content_item_id")
private ContentItem contentItem;
```

Это создает сквозную зависимость по JPA:

- `courses-service` становится зависим от `content-service`;
- нельзя запустить сервис без наличия структуры чужой БД;
- нарушается граница между доменами;

## Как правильно

Вместо этого мы просто храним внешний идентификатор:

```java
private UUID contentItemId;
```

И при необходимости получаем информацию о контенте через Feign:

```java
@FeignClient(name = "content-service")
public interface ContentClient {

    @GetMapping("/api/content/{id}")
    ContentItemDTO getById(@PathVariable UUID id);
}
```

Таким образом:

- база `courses-service` остается чистой и независимой;
- никаких внешних JOIN;
- взаимодействие идет через стабильный API.

## Каждый сервис — отдельный мир

Это касается не только контента. Например, `profiles-service` управляет пользователями,
`courses-service` может ссылаться на `profileId`, но не должен знать, как устроена таблица профилей.

Если нужно получить данные профиля — используем `ProfileClient` с Feign:

```java
@FeignClient(name = "profiles-service")
public interface ProfileClient {

    @GetMapping("/api/profiles/{id}")
    ProfileDTO getProfile(@PathVariable UUID id);
}
```

## Принципы слабой связанности

|Принцип                         |	Почему это важно                   |
|--------------------------------|-------------------------------------|
|У каждого сервиса — своя БД     |	Независимость, масштабируемость    |
|Только DTO между сервисами      |	Четкие границы, простота изменений | 
|JPA-связи только внутри сервиса |	Контроль и модульность             |
|REST/Feign между сервисами      |	Гибкость, устойчивость             | 

## Статьи серии

- **[Микросервисы: серия материалов о принципах, паттернах и практике](/posts/mservices-intro/)**
- **[Эволюция архитектур: от монолита к микросервисам](/posts/mservices-evolution/)**
- **[Микросервисная архитектура](/posts/mservices-architecture/)**
- **[Нужен ли Service Discovery в Docker-среде?](/posts/service-discovery-needed/)**
- **[Вызов других микросервисов с помощью Feign](/posts/open-feign-client-intro/)**
- **[Как работает Service Discovery в Spring Cloud и зачем он нужен](/posts/service-discovery/)**
- _(в разработке)_
