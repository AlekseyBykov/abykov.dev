---
layout: post
title: "Эволюция Java: что добавлялось и что удалялось"
date: 2025-09-17
categories: [java, jvm]
tags: [java, evolution, jvm, history, api]
---

Java появилась в 1996 году и за почти 30 лет превратилась из простого объектно-ориентированного языка в мощную платформу с современными возможностями. С каждым крупным релизом в Java добавлялись новые фичи и API, а устаревшие подходы постепенно уходили в прошлое.

В этой статье рассмотрим **ключевые изменения языка и платформы**, начиная с Java 5 и заканчивая Java 21:  
- Какие возможности появились?
- Что упростилось для разработчиков?
- От каких технологий пришлось отказаться?

## Java 5 (2004)

До 2004 года Java развивалась в основном библиотеками и API, но язык оставался почти неизменным.  
Версия 5 стала настоящей революцией: появились дженерики, аннотации, перечисления, улучшения работы с потоками и автозамена типов.  

### Generics

До Java 5 коллекции работали с `Object`, что приводило к постоянным кастам и ошибкам в runtime.  

```java
// До Java 5
List list = new ArrayList();
list.add("hello");
Integer num = (Integer) list.get(0); // ClassCastException
```
С дженериками мы получили **типобезопасность на этапе компиляции**.
```java
// С Java 5
List<String> list = new ArrayList<>();
list.add("hello");
String str = list.get(0); // безопасно
```

### PECS (Producer Extends, Consumer Super)

Вместе с дженериками появились wildcards (`?`, `? extends`, `? super`). Они нужны, чтобы писать универсальные методы.

Пример из `Collections.copy`:
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {
    for (int i = 0; i < src.size(); i++) {
        dest.set(i, src.get(i));
    }
}
```

Правило **PECS** (*Effective Java, Джошуа Блох*):

- **Producer Extends** — если коллекция только отдает элементы (producer), используем `? extends T`.
- **Consumer Super** — если коллекция принимает элементы (consumer), используем `? super T`.

Наглядный пример:
```java
// Producer: ? extends Number
List<? extends Number> numbers = List.of(1, 2, 3.14);

// numbers.add(10);        // нельзя добавлять (тип не гарантирован)
Number n = numbers.get(0); // можно читать как Number или Object

// Consumer: ? super Integer
List<? super Integer> integers = new ArrayList<Number>();

integers.add(10);             // можно добавлять Integer и его сабклассы
Object obj = integers.get(0); // читаем только как Object
```

- `? extends Number` хранит **подтипы Number** (`Integer`, `Double` и тд), но мы можем их только читать, не добавлять.
- `? super Integer` может хранить **Integer и его подтипы**, но при чтении получаем только `Object`.

Без wildcard-типов дженерики были бы куда менее гибкими.

### Аннотации

Раньше для метаданных использовались комментарии `@deprecated` или XML-конфигурации. С Java 5 появились полноценные аннотации.
```java
@Override
public String toString() {
    return "Hello";
}
```

Аннотации открыли дорогу к Spring, Hibernate и JUnit в современном виде — конфигурация из XML постепенно уходит.

### Enum

До этого программисты эмулировали перечисления через int-константы. Это было небезопасно:
```java
// Старый стиль
public static final int MONDAY = 1;
public static final int TUESDAY = 2;

int day = 5; // валидно, но смысла нет
```

Перечисления (enum) решили проблему:
```java
enum Day { MONDAY, TUESDAY, WEDNESDAY }
Day day = Day.MONDAY;
```

### Autoboxing / Unboxing

До Java 5 нужно было явно оборачивать примитивы в объекты и доставать их обратно:

```java
// До Java 5
Integer num = new Integer(5);
int n = num.intValue();
```

С Java 5 это стало происходить автоматически:
```java
Integer num = 5;  // autoboxing
int n = num;      // unboxing
```

### Varargs

Вместо передачи массива вручную можно объявить метод с переменным числом аргументов:

```java
// До Java 5
void printAll(String[] args) { ... }
printAll(new String[]{"a", "b", "c"});

// С Java 5
void printAll(String... args) { ... }
printAll("a", "b", "c");
```

### Static import

Появилась возможность импортировать статические методы и поля, чтобы вызывать их без класса:
```java
import static java.lang.Math.*;

double r = sqrt(PI); // вместо Math.sqrt(Math.PI)
```

### Улучшения Concurrency API

Ранее, в Java 1.0–1.4, были:

- Класс `Thread`;
- Ключевое слово `synchronized`;
- Методы `wait()`, `notify()`, `notifyAll()`;
- Примитивы вроде `Thread.stop()` / `Thread.suspend()` (потом признаны небезопасными и deprecated).

Это был очень низкоуровневый и небезопасный способ. В Java 5 в стандартную библиотеку добавили пакет `java.util.concurrent`, где появились:

- `Executor` и `ExecutorService` — абстракция над пулом потоков;
- `Callable` и `Future` — возможность вернуть результат из потока;
- `ConcurrentHashMap` и другие thread-safe коллекции;
- Синхронизаторы: `CountDownLatch`, `Semaphore`, `CyclicBarrier`, `BlockingQueue`;
- `ReentrantLock`, `ReadWriteLock`, `Condition` — альтернатива synchronized.

То есть Java 5 — это первая версия, где многопоточность стала «практичной» для приложений:

```java
ExecutorService pool = Executors.newFixedThreadPool(2);

Future<Integer> result = pool.submit(new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
        // Выполним какую-то задачу
        return 42;
    }
});

try {
    System.out.println("Результат: " + result.get()); // блокирующее получение
} catch (Exception e) {
    e.printStackTrace();
} finally {
    pool.shutdown();
}
```


### Deprecated и удаленное

В этой версии ничего критичного не удалили, но методы `Thread.stop()`, `Thread.suspend()`, `Thread.resume()` получили официальный статус **deprecated** (опасны для использования). Некоторые методы AWT (графическая библиотека) также были объявлены устаревшими.

### Итог по Java 5

| Feature      | Added                                                                 | Deprecated/Removed |
|--------------|----------------------------------------------------------------------|--------------------|
| Generics     | Параметризованные типы<br>Wildcards (PECS)                           | — |
| Annotations  | `@Override`<br>`@Deprecated`<br>`@SuppressWarnings`                  | — |
| Enums        | Перечисления как полноценный тип                                     | — |
| Syntax sugar | Autoboxing / Unboxing<br>Varargs (`...`)<br>Static import            | — |
| Concurrency  | Новый пакет `java.util.concurrent`<br>Executors<br>Future<br>ConcurrentMap | `Thread.stop()`<br>`Thread.suspend()`<br>`Thread.resume()` |
| AWT          | —                                                                    | Некоторые методы объявлены deprecated |

**Почему это важно**: Java 5 заложила основу для современного языка. Без дженериков мы бы не имели Streams API и Optional,а без enum — читаемости и безопасности в моделях. Аннотации стали фундаментом для Spring и JUnit. Autoboxing, varargs и static import упростили повседневный код, Concurrency API дало разработчикам мощные инструменты для многопоточности.


