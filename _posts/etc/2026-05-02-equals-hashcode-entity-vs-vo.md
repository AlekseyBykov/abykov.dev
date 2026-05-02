---
layout: post
title: "equals() и hashCode(): Entity vs Value Object"
date: 2026-05-02
categories: [java, persistence]
tags: [equals, hashcode, jpa, hibernate, domain-model]
---

Реализация `equals()` и `hashCode()` для JPA-сущностей отличается от обычных объектов: для сущностей равенство, как правило, определяется по одному полю — идентификатору (`id`).

На первый взгляд это выглядит как упрощение ради производительности: сравнение по одному полю действительно проще и быстрее. Однако это лишь побочный эффект. Основная причина — различие в том, что сравнивается: состояние объекта или его идентичность.

Это становится критичным, когда объект используется в `HashSet` или `HashMap`, а также в процессе работы ORM, где сущность может изменяться в течение своего жизненного цикла.

### Value Object: равенство по состоянию

Value Object — это объект, у которого нет собственного идентификатора. Его равенство определяется значениями полей:

```java
public class Ingredient {

    private String name;
    private Integer calories;

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof Ingredient)) {
            return false;
        }
        Ingredient that = (Ingredient) o;
        return Objects.equals(name, that.name)
                && Objects.equals(calories, that.calories);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, calories);
    }
}
```

```java
new Ingredient("cheese", 200)
        .equals(new Ingredient("cheese", 200)); // true
```

Такой подход работает, потому что объект полностью задатся своими полями. Если значения совпадают, объекты считаются равными.

### Entity: равенство по идентичности

Сущность — это не просто набор данных, а ссылка на запись в базе. Поэтому важно не содержимое полей, а то, указывают ли два объекта на одну и ту же запись.

```text
MenuItem(id=1, name="Pizza", price=10)
MenuItem(id=1, name="Pizza", price=12)
```

Данные отличаются, но это одна и та же сущность. Поэтому равенство определяется по `id`:

```java
@Entity
public class MenuItem {

    @Id
    @GeneratedValue
    private Long id;

    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        if (!(o instanceof MenuItem)) {
            return false;
        }
        MenuItem that = (MenuItem) o;
        return id != null && id.equals(that.id);
    }

    @Override
    public int hashCode() {
        return getClass().hashCode();
    }
}
```

#### Что ломается при сравнении по полям

Проблема проявляется, когда сущность меняется в процессе работы:

```java
Set<MenuItem> set = new HashSet<>();
MenuItem item = new MenuItem();

set.add(item);
entityManager.persist(item); // появляется id

set.contains(item); // может вернуть false
```

После `persist()` у объекта появляется `id`. Это меняет результат `equals()`, но само по себе не является проблемой.

Причина в том, как работают коллекции:
* сначала используется `hashCode()` — чтобы найти корзину
* затем вызывается `equals()` — чтобы проверить совпадение

Если `hashCode()` не меняется, объект остатся в той же корзине и корректно находится. Изменение `equals()` в этом случае не нарушает работу коллекции.

Проблема возникает только тогда, когда меняется `hashCode()`. Если `hashCode()` зависит от `id` или других изменяемых полей, он тоже изменится. В этом случае коллекция больше не сможет найти объект.

Поэтому:
* `equals()` может использовать `id`
* `hashCode()` должен оставаться стабильным

Именно по этой причине `hashCode()` не зависит от `id`.

#### Прокси Hibernate

При работе с ORM объект сущности не всегда представлен своим реальным классом.

Hibernate использует прокси для ленивой загрузки, поэтому вместо сущности может быть возвращен объект-обертка с тем же идентификатором. Это происходит не во всех случаях, но является нормальным и ожидаемым поведением.


```java
MenuItem proxy = entityManager.getReference(MenuItem.class, 1L);
```

Такой объект имеет другой runtime-класс, делегирует вызовы реальной сущности и может быть не полностью инициализирован. Из-за этого проверка через `getClass()` может давать неверный результат:

```java
if (getClass() != o.getClass()) {
    return false; // может не сработать корректно
}
```

Более простой и распространенный вариант — использовать `instanceof`, который корректно обрабатывает прокси:

```java
if (!(o instanceof MenuItem)) {
    return false;
}
```

#### Учет прокси при сравнении

В более точной реализации учитывается реальный класс сущности, а не класс прокси.

```java
@Override
public boolean equals(Object o) {
    if (this == o) {
        return true;
    }

    if (o == null || resolvePersistentClass(this) != resolvePersistentClass(o)) {
        return false;
    }

    MenuItem that = (MenuItem) o;
    return id != null && id.equals(that.id);
}

@Override
public final int hashCode() {
    return resolvePersistentClass(this).hashCode();
}

private static Class<?> resolvePersistentClass(Object o) {
    return o instanceof HibernateProxy
            ? ((HibernateProxy) o).getHibernateLazyInitializer().getPersistentClass()
            : o.getClass();
}
```

Здесь:

* сравнение идет по реальному классу сущности
* прокси и обычный объект считаются одним типом
* `hashCode()` остается стабильным

Такой подход избавляет от проблем, связанных с прокси, и соответствует рекомендациям для сущностей с генерируемым идентификатором.

#### Где тут часто ошибаются

Главная ловушка — использовать генерацию через Lombok:

```java
@Data // ❌
@Entity
public class MenuItem { ... }
```

`@Data` включает все поля в `equals()` и `hashCode()`, что приводит к описанным выше проблемам.

### Итог

Различие простое, но принципиальное:

```text
Value Object → равенство по состоянию
Entity       → равенство по идентичности (id)
```

Это не рекомендация, а требование корректной работы ORM и коллекций.
