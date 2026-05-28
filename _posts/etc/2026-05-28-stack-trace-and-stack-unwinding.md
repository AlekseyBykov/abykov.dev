---
layout: post
title: "Stack trace и stack unwinding"
date: 2026-05-28
categories: [java, jvm]
tags: [java, jvm, exceptions, stacktrace, hotspot]
---

**stack trace** выглядит как обычный текст ошибки, однако за ним стоит довольно сложная модель обработки. Во время создания исключения, JVM проходит по фреймам стека, формирует его снимок, создает `StackTraceElement[]`, сохраняет снимок внутри объекта исключения, выполняет размотку стека.

Поэтому исключения в Java — это не просто `throw`, а полноценная операция взаимодействия с внутренней моделью выполнения JVM.

## Stack вызовов

Когда в Java происходит исключение, JVM печатает stack trace:

```text
Exception in thread "main" java.lang.ArithmeticException: / by zero
    at Example.calculate4(Example.java:19)
    at Example.calculate3(Example.java:15)
    at Example.calculate2(Example.java:11)
    at Example.calculate1(Example.java:7)
    at Example.main(Example.java:3)
```

Обычно stack trace воспринимается просто как текст ошибки. На практике это отображение цепочки вызовов методов, которая привела программу к исключению.

Во время выполнения JVM хранит вызовы методов в стеке, который работает по принципу:

```text
Last In → First Out
```

Последним в стек попадает метод, который выполняется прямо сейчас, и именно он покидает его первым после завершения работы. Т.е. при вызове метода JVM создает новый фрейм и помещает его на вершину стека.

У каждого потока JVM существует собственный стек. JVM использует его для хранения информации о выполняющихся методах текущего потока. Каждый вызов метода создает внутри стека отдельный фрейм (**stack frame**).

Упрощенно фрейм содержит:

* параметры метода;
* локальные переменные;
* промежуточные результаты вычислений;
* информацию о возврате из метода.

Когда метод завершается, его фрейм удаляется из стека, а управление возвращается в предыдущий метод.

Например, если один метод вызывает другой:

```java
main()
  ↓
calculate1()
  ↓
calculate2()
  ↓
calculate3()
```

то в стеке потока в этот момент будет находиться примерно такая цепочка фреймов:

```text
Thread Stack
-------------
calculate3()
calculate2()
calculate1()
main()
```

Последним в стеке всегда находится метод, который выполняется прямо сейчас.

## Stack trace и StackTraceElement[]

Когда происходит исключение, JVM получает текущее состояние стека и создает его снимок (**snapshot**) — объектное представление стека вызовов на момент ошибки. Этот снимок сохраняется внутри исключения в виде массива:

```java
StackTraceElement[]
```

Упрощенно это можно представить следующим образом. Во время выполнения программы стек потока выглядит так:

```text
Thread Stack
-------------
calculate4()
calculate3()
calculate2()
calculate1()
main()
```

После возникновения исключения JVM создает снимок стека:

```text
Heap
-------------
Exception
  -> StackTraceElement[]
        [0] calculate4()
        [1] calculate3()
        [2] calculate2()
        [3] calculate1()
        [4] main()
```

Именно этот снимок затем выводится в виде stack trace. Поэтому **stack trace — это объектное представление стека** на момент возникновения ошибки.

## Пример exception

Рассмотрим простой пример:

```java
public class Example {

    public static void main(String[] args) {
        calculate1();
    }

    public static void calculate1() {
        calculate2();
    }

    public static void calculate2() {
        calculate3();
    }

    public static void calculate3() {
        calculate4();
    }

    public static void calculate4() {
        System.out.println(10 / 0);
    }
}
```

Во время выполнения `calculate4()` цепочка вызовов внутри стека потока будет выглядеть следующим образом:

```text
Thread Stack
-------------
calculate4()
calculate3()
calculate2()
calculate1()
main()
```

В строке:

```java
System.out.println(10 / 0);
```

JVM генерирует `ArithmeticException`. В этот момент происходит не просто создание объекта исключения. JVM:

* получает текущее состояние стека потока;
* проходит по фреймам стека;
* формирует снимок цепочки вызовов;
* преобразует его в массив `StackTraceElement[]`;
* сохраняет этот снимок внутри объекта исключения.

Упрощенно это можно представить так:

```text
Heap
-------------
ArithmeticException
    ↓
StackTraceElement[]
    [0] calculate4()
    [1] calculate3()
    [2] calculate2()
    [3] calculate1()
    [4] main()
```

Именно это объектное представление стека затем выводится в виде stack trace:

```text
Exception in thread "main" java.lang.ArithmeticException: / by zero
    at Example.calculate4(Example.java:19)
    at Example.calculate3(Example.java:15)
    at Example.calculate2(Example.java:11)
    at Example.calculate1(Example.java:7)
    at Example.main(Example.java:3)
```

В первой строке JVM выводит:

* поток, в котором произошло исключение;
* тип исключения;
* сообщение ошибки.

Далее идет история вызовов методов — от места возникновения ошибки к исходной точке входа программы.

Для каждого вызова JVM показывает:

* класс;
* метод;
* файл;
* номер строки.

Поскольку стек работает по принципу LIFO, первым в stack trace отображается метод, в котором произошло исключение, а последним — `main()`.

## Stack unwinding

После создания исключения JVM начинает подниматься вверх по фреймам стека в поисках подходящего `catch`. Этот процесс называется stack unwinding — размотка стека вызовов. JVM начинает последовательно удалять фреймы:

```text
calculate4()
↑
calculate3()
↑
calculate2()
↑
calculate1()
↑
main()
```

До тех пор, пока либо не найдет подходящий `catch`, либо поток не завершится аварийно. Здесь становится понятно, почему исключения считаются относительно дорогой операцией.

## fillInStackTrace()

До этого момента исключения можно было воспринимать как обычный Java-объект, внутри которого хранится снимок стека вызовов. Однако внутри JVM создание исключения — довольно тяжелая операция.

Ключевую роль здесь играет метод:

```java
fillInStackTrace()
```

Именно он отвечает за формирование stack trace и создание массива `StackTraceElement[]`. Упрощенно создание исключения внутри `Throwable` выглядит примерно так:

```java
public Throwable() {
    fillInStackTrace();
}
```

> ⚠️ Это означает, что при создании практически любого исключения JVM автоматически начинает собирать информацию о текущем состоянии 
> стека потока.

Во время выполнения `fillInStackTrace()` JVM:

* получает текущее состояние стека потока;
* проходит по фреймам;
* извлекает метаданные методов;
* определяет номера строк;
* формирует массив `StackTraceElement[]`;
* сохраняет его внутри исключения.

Важно понимать, что `fillInStackTrace()` — это не просто обычный Java-метод. Внутри HotSpot JVM его выполнение связано с внутренними моделями виртуальной машины, работающими с фреймами напрямую. Упрощенно процесс выглядит примерно так:

```text
Throwable
    ↓
fillInStackTrace()
    ↓
JVM runtime
    ↓
walk thread stack
    ↓
create StackTraceElement[]
```

Именно поэтому исключения в Java — это значительно больше, чем просто создание объекта через `new`, особенно если исключения возникают массово или внутри hot-path логики. Например, `new Object()` и `new RuntimeException()` имеют принципиально разную стоимость. В первом случае JVM просто выделяет память под объект.

## Lightweight exceptions

Поскольку основная стоимость исключений связана с формированием stack trace, в некоторых случаях JVM или фреймворки стараются избежать этой операции. Для этого используются так называемые **lightweight exceptions** — исключения без полноценного stack trace.

Один из способов реализовать это — переопределить метод:

```java
fillInStackTrace()
```

Например:

```java
public class FastException extends RuntimeException {

    @Override
    public synchronized Throwable fillInStackTrace() {
        return this;
    }
}
```

В этом случае JVM не будет проходить по фреймам стека, формировать снимок вызовов, создавать `StackTraceElement[]`. Фактически исключение превращается почти в обычный объект. Это особенно важно в hot-path коде, где исключения могут создаваться очень часто.

Например:

* внутри сетевых фреймворков;
* reactive runtime;
* low-level infrastructure;
* высоконагруженных pipeline.

Однако у такого подхода есть серьезный недостаток. Поскольку stack trace не создается, исключение перестает содержать информацию о месте возникновения ошибки.

Например, `e.printStackTrace()` выведет либо пустой stack trace, либо сильно урезанный. Из-за этого lightweight exceptions редко используются в прикладном коде и обычно встречаются только внутри фреймворков или в runtime-инфраструктуре.

Начиная с Java 7 появился и более официальный способ отключения stack trace через конструктор `Throwable`:

```java
Throwable(
    String message,
    Throwable cause,
    boolean enableSuppression,
    boolean writableStackTrace
)
```

Например:

```java
new RuntimeException(
    null,
    null,
    false,
    false
);
```

Если параметр `writableStackTrace = false`, то JVM не будет вызывать `fillInStackTrace()` и создавать `StackTraceElement[]`.

Таким образом, стоимость исключения в Java напрямую связана не с самим `throw`, а с формированием stack trace.

## Caused by и цепочки исключений

В простых примерах stack trace обычно содержит одно исключение. Однако в реальных приложениях ошибки часто заворачиваются друг в друга по мере прохождения через разные слои системы.

Рассмотрим пример:

```java
public class Example {

    public static void main(String[] args) {
        service();
    }

    static void service() {
        try {
            repository();
        } catch (IllegalStateException e) {
            throw new RuntimeException("Service failed", e);
        }
    }

    static void repository() {
        throw new IllegalStateException("DB offline");
    }
}
```

Здесь `repository()` создает исходное исключение:

```java
throw new IllegalStateException("DB offline");
```

В этот момент стек выглядит следующим образом:

```text
repository()
service()
main()
```

JVM формирует снимок этого состояния и сохраняет его внутри `IllegalStateException`. Однако затем exception перехватывается:

```java
catch (IllegalStateException e)
```

и создается новое исключение:

```java
throw new RuntimeException("Service failed", e);
```

И это уже второй объект исключения со своим собственным stack trace. К этому моменту фрейм метода `repository()` уже удален из стека во время stack unwinding, поэтому новый снимок будет выглядеть так:

```text
service()
main()
```

В результате образуется цепочка исключений:

```text
RuntimeException
    ↓ cause
IllegalStateException
```

JVM выводит это следующим образом:

```text
java.lang.RuntimeException: Service failed
    at Example.service(Example.java:11)
    at Example.main(Example.java:4)

Caused by: java.lang.IllegalStateException: DB offline
    at Example.repository(Example.java:17)
    at Example.service(Example.java:9)
    at Example.main(Example.java:4)
```

Строка `Caused by:` означает переход к предыдущему исключению в цепочке. Внутри `Throwable` для этого существует специальное поле:

```java
Throwable cause
```

Именно поэтому при анализе сложных stack trace обычно ищут:

* самый глубокий `Caused by`;
* первое meaningful exception;
* место первоначального возникновения ошибки.

## common frames omitted

В предыдущем примере можно заметить, что часть stack trace повторяется:

```text
at Example.service(...)
at Example.main(...)
```

Эти фреймы присутствуют и в `RuntimeException`, и в `IllegalStateException`. Чтобы не дублировать одинаковые фреймы, JVM сокращает вывод:

```text
java.lang.RuntimeException: Service failed
    at Example.service(Example.java:11)
    at Example.main(Example.java:4)

Caused by: java.lang.IllegalStateException: DB offline
    at Example.repository(Example.java:17)
    ... 2 common frames omitted
```

Строка:

```text
... 2 common frames omitted
```

означает, что нижние два фрейма уже были выведены выше и совпадают с предыдущим stack trace.

Поэтому в больших enterprise-приложениях stack trace часто выглядит как цепочка связанных exception, распространяющихся через разные уровни системы.

## Exceptions как anti-pattern

Поскольку создание исключений связано с обходом фреймов стека и формированием stack trace, исключения считаются относительно дорогой операцией. По этой причине их обычно не используют как способ управления потоком выполнения.

Например, такой код может выглядеть компактно и удобно:

```java
try {
    Integer.parseInt(value);
    return true;
} catch (NumberFormatException e) {
    return false;
}
```

Однако если подобная логика выполняется массово — например, внутри циклов, parser pipeline или high-load обработки данных — JVM будет постоянно:

* создавать исключения;
* проходить по фреймам стека;
* формировать снимки стека;
* создавать `StackTraceElement[]`;
* выполнять размотку.

В результате большое количество времени и памяти начинает тратиться не на бизнес-логику, а на инфраструктурную обработку исключений. Поэтому исключения в Java служат для обработки действительно исключительных ситуаций:

* ошибок ввода-вывода;
* проблем сети;
* некорректного состояния приложения;
* нарушения контрактов API;
* неожиданных ошибок выполнения.

Но не для обычной логики ветвления.

Например, вместо:

```java
try {
    Integer.parseInt(value);
} catch (NumberFormatException e) {
    return false;
}
```

предпочтительнее явная проверка входных данных:

```java
value.matches("\\d+")
```

или использование специализированных parser API.

Разумеется, современные JVM умеют хорошо оптимизировать работу с исключениями, а в обычном прикладном коде разница часто оказывается несущественной. Однако понимание того, что происходит внутри JVM при создании исключений, помогает лучше понимать:

* стоимость stack trace;
* природу stack unwinding;
* причины появления lightweight exceptions;
* почему исключения считаются дорогими.
