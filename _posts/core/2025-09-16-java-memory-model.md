---
layout: post
title: "Java Internals: Java Memory Model простыми словами"
date: 2025-09-16
categories: [java, concurrency]
tags: [java, jvm, jmm, multithreading, volatile, synchronized]
---

Когда мы пишем многопоточный код на Java, часто сталкиваемся с неожиданным поведением: значение переменной вдруг не обновляется, потоки работают с устаревшими данными, а `synchronized` или `volatile` ведут себя неочевидно.  

Все это связано с тем, как **JVM управляет памятью в многопоточной среде**.  

Чтобы программист не зависел от конкретного процессора или архитектуры, была создана спецификация **Java Memory Model (JMM)**.  

JMM отвечает на вопросы:
- Как потоки обмениваются данными через память?
- В каком порядке могут выполняться инструкции (и что реально увидят другие потоки)?
- Что гарантируют ключевые слова `volatile`, `synchronized`, `final` и `atomic`?
- Почему оптимизации процессора и компилятора не должны ломать корректность кода?

## Видимость (Visibility)

### Проблема: невидимые изменения

Начнем с простого примера:

```java
public class VisibilityDemo {

    private static boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            while (running) {
                // пустой цикл
            }
            System.out.println("Thread stopped");
        });

        t.start();
        Thread.sleep(1000);

        running = false; // ожидаем, что поток завершится
        System.out.println("Main updated running=false");
    }
}
```

Ожидается, что через секунду переменная `running` изменится на `false`, и поток завершится. Однако поток зависает бесконечно, вывод `"Thread stopped"` так и не появляется.

Почему? Потому что у каждого потока может быть свой кеш переменной, и без дополнительных гарантий изменения из main-потока не станут видны рабочему потоку.

Это первая фундаментальная проблема JMM — **видимость (visibility)**.

Чтобы это исправить, в Java есть ключевое слово **`volatile`**:

### Решение через volatile

```java
public class VisibilityDemo {

    private static volatile boolean running = true;

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            while (running) {
                // теперь поток будет видеть изменения
            }
            System.out.println("Thread stopped");
        });

        t.start();
        Thread.sleep(1000);

        running = false; // обновление гарантированно станет видно
        System.out.println("Main updated running=false");
    }
}
```
Теперь второй поток корректно завершится.

Что гарантирует `volatile`:

1. **Все чтения идут из общей памяти, а не из кеша потока**.
- Каждый раз при чтении переменной поток идет в main memory (heap), а не использует локальные копии.
- Каждое обновление сразу пишется в main memory.

2. **Порядок операций (happens-before)**.
- Запись в volatile переменную всегда видна всем последующим чтениям этой переменной из других потоков.
- Это гарантирует, что изменения проезжают по памяти в нужном порядке.

### Ограничения volatile

`volatile` гарантирует **видимость**, но не **атомарность**. Если несколько потоков одновременно изменяют значение (`counter++`), результат будет некорректным, потому что операция `++` = *"прочитать → увеличить → записать"* состоит из нескольких шагов.

Пример:
```java
public class VolatileCounter {
    
    private static volatile int counter = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter++;
            }
        });

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                counter++;
            }
        });

        t1.start();
        t2.start();

        t1.join();
        t2.join();

        System.out.println("Result: " + counter);
    }
}
```

**Ожидаем** `2000`, **на практике** меньше (например, `1673`).

Почему так?

- `counter++` раскладывается в байткод: `load` → `add` → `store`.
- Между этими шагами могут вклиниться потоки и обновления будут потеряны.
- `volatile` не спасает, тк. оно лишь гарантирует видимость, но не делает всю операцию атомарной.

Для атомарных операций нужны:

- `synchronized` блоки;
- Классы из `java.util.concurrent.atomic` (`AtomicInteger` и др.).

## Атомарность (Atomicity)

**Атомарность** — это гарантия, что операция выполняется "целиком или никак". В многопоточных приложениях это критично: если операция состоит из нескольких шагов (как `counter++`), другой поток не должен вклиниться между ними.

В Java атомарность обеспечивают:
1. `synchronized` — блокировки на уровне монитора объекта.
2. `Lock` API из `java.util.concurrent.locks` (например, `ReentrantLock`).
3. Специальные атомарные классы (`AtomicInteger`, `AtomicLong` и т.д.).

### synchronized

```java
public class SyncCounter {

    private static int counter = 0;

    public static synchronized void increment() {
        counter++;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                increment();
            }
        });
        
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) {
                increment();
            }
        });

        t1.start();
        t2.start();
        
        t1.join();
        t2.join();

        System.out.println("Result: " + counter);
    }
}
```
Теперь вывод всегда будет: `Result: 2000`

Почему?

- `synchronized` гарантирует, что метод `increment()` одновременно выполняется только в одном потоке.
- Другие потоки ждут, пока монитор (блокировка) освободится.

`synchronized` решает сразу две задачи:

1. **Взаимное исключение (mutual exclusion)** — атомарность доступа к блоку/методу.
2. **Видимость (visibility)** — при выходе из синхронизированного блока поток сбрасывает все изменения в память, и другие потоки их увидят.

То есть это решение "все в одном", в отличие от `volatile`.

### ReentrantLock

Для большей гибкости можно использовать `ReentrantLock`:
```java
import java.util.concurrent.locks.ReentrantLock;

public class LockCounter {
    
    private static int counter = 0;
    private static final ReentrantLock lock = new ReentrantLock();
    
    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) {
                lock.lock();
                try {
                    counter++;
                } finally {
                    lock.unlock();
                }
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);
        
        t1.start();
        t2.start();
        
        t1.join();
        t2.join();

        System.out.println("Result: " + counter);
    }
}
```
`ReentrantLock` дает больше контроля:

- Можно проверять, доступна ли блокировка (`tryLock`);
- Можно делать блокировки с таймаутом;
- Можно использовать `Condition` для реализации более сложных сценариев (например, producer-consumer).

### Итоги

- `volatile` = видимость (но не атомарность).
- `synchronized` = и атомарность, и видимость.
- Для счетчиков и простых операций лучше использовать `AtomicInteger`.
- Для сложных сценариев — `Lock`, `ReadWriteLock`, `StampedLock`.

## Порядок выполнения (Reordering) и happens-before

Компилятор и процессор могут **менять порядок инструкций**, если это не меняет результат в однопоточном коде. Но в многопоточном коде это приводит к сюрпризам: другой поток может увидеть переменные в неожиданном состоянии.

Пример:

```java
int a = 0, b = 0;
int x = 0, y = 0;

Thread t1 = new Thread(() -> {
    a = 1;        // (1)
    x = b;        // (2)
});

Thread t2 = new Thread(() -> {
    b = 1;        // (3)
    y = a;        // (4)
});

t1.start(); 
t2.start();

t1.join(); 
t2.join();

System.out.println("x=" + x + ", y=" + y);
```
Что ожидаем:

- либо (`x=0, y=1`),
- либо (`x=1, y=0`),
- либо (`x=1, y=1`).

Что может произойти на практике: `(x=0, y=0)` — из-за reorder процессор может переставить строки (1) и (2), а также (3) и (4).

JMM вводит специальное правило: **happens-before**. Если событие **A happens-before B**, то все изменения, сделанные в A, будут видны в B.

Здесь `t1.join(); t2.join();` гарантирует только то, что main-поток дождется завершения `t1` и `t2`. Но внутри самих потоков никаких happens-before между (1) и (2) → (3) и (4) нет, поэтому `(x=0, y=0)` формально допустимо.

### Основные случаи happens-before

JMM дает набор правил, которые говорят компилятору и процессору: *"эти действия нельзя переставлять местами, и результаты должны быть видны другим потокам"*. Это не про Java как язык, а именно про **барьеры оптимизаций и кешей**. По сути, это набор синхронизационных точек, с которыми процессор и компилятор обязаны считаться. Без них они могли бы переставить инструкции и сломать логику многопоточного кода.

#### Program order rule

В одном потоке действия выполняются в порядке программы. Если написать:
```java
a = 1;
b = 2;
```
То поток гарантированно сначала присвоит `a=1`, потом `b=2`, но для других потоков это не гарантируется. Чтобы они увидели изменения в правильном порядке, нужны `volatile`, `synchronized` или другие правила.

#### Monitor lock rule

Unlock на мониторе happens-before последующему lock на том же мониторе. Пример:
```java
synchronized(lock) { shared = 10; } // unlock произойдет при выходе
synchronized(lock) { System.out.println(shared); } // lock
```
Второй блок обязательно увидит 10, потому что выход из первого синхронизированного блока сбрасывает изменения в память.

#### Volatile variable rule

Запись в `volatile` переменную happens-before ее чтению. Пример:
```java
volatile boolean flag = false;

flag = true;   // запись
if (flag) { ... } // чтение
```

#### Thread start rule

Вызов `Thread.start()` на потоке `T` happens-before любому действию в потоке `T`.
```java
t.start();  // гарантировано доходит до run()
```

#### Thread termination rule

Все действия в потоке `T` happen-before успешному возврату из `T.join()` в другом потоке.
```java
t.join(); // // гарантирует, что все из потока t завершилось
```

#### Interruption rule

Вызов `t.interrupt()` happens-before моменту, когда поток замечает прерывание (через `isInterrupted()` или `InterruptedException`).

#### Finalizer rule

Конструктор объекта happens-before запуском его `finalize()` (если вдруг используется финализатор).

#### Transitivity rule

Если A happens-before B, а B happens-before C → значит A happens-before C.

### Double-checked locking (DCL)

Пример, где без happens-before все ломается:
```java
class Singleton {
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

Почему нужен `volatile`? Поскольку `new Singleton()` на байткоде может быть развернута в:
- Выделение памяти (1);
- Присвоение ссылки `instance` (2);
- Вызов конструктора (3).

JMM может переставить шаги 2 и 3. В итоге другой поток может увидеть `instance != null`, но объект еще не проинициализирован.

С `volatile` запись в `instance` happens-before ее чтения, и объект гарантированно корректен.

### Итоги по reordering

- Компилятор и CPU могут менять порядок инструкций.
- JMM задает четкие правила happens-before.
- `volatile` и `synchronized` создают барьеры памяти, которые запрещают reorder.
- Если в коде нет отношений happens-before — результат непредсказуем.

## Final и конструкторы

У `final` полей в Java есть **особая гарантия видимости**: если объект корректно сконструирован и ссылка на него передана другому потоку, то этот поток гарантированно увидит правильные значения `final` полей.

Пример:
```java
class Holder {
    final int value;

    Holder(int v) {
        this.value = v;
    }
}

public class FinalDemo {
    static Holder holder;

    public static void main(String[] args) throws InterruptedException {
        Thread writer = new Thread(() -> {
            holder = new Holder(22); // конструктор завершен
        });

        Thread reader = new Thread(() -> {
            while (holder == null) {}
            System.out.println(holder.value); // всегда 22
        });

        writer.start();
        reader.start();

        writer.join();
        reader.join();
    }
}
```
Здесь поток `reader` всегда напечатает `22`. Даже если нет `volatile` или `synchronized`, JMM гарантирует, что `final` поля будут корректно видны.

### Когда это ломается

Если объект утекает до завершения конструктора, гарантии теряются:
```java
class BrokenHolder {
    final int value;

    static BrokenHolder leaked;

    BrokenHolder(int v) {
        leaked = this;     // утечка "this" из конструктора
        this.value = v;
    }
}
```
Теперь другой поток может получить ссылку на объект через `leaked`, но его поле `value` еще не инициализировано. Результат: `0` вместо ожидаемого значения.

### Вывод

- `final` в Java играет роль не только "нельзя поменять", но и добавляет гарантию видимости.
- Если объект создан корректно (без утечки `this`), то другие потоки увидят `final` поля проинициализированными.
- Если объект утекает в процессе конструктора, то никакой гарантии нет.


## CAS и атомарные классы

До сих пор мы говорили про `volatile` и `synchronized`. Но начиная с **Java 5** в стандартной библиотеке появился пакет `java.util.concurrent.atomic`, который предлагает более быстрые неблокирующие примитивы.

В их основе лежит техника **CAS (compare-and-swap)**.

### Что такое CAS

CAS — это атомарная операция на уровне процессора. Она делает следующее:

```text
compare-and-swap(адрес, ожидаемое_значение, новое_значение)
```

- если по адресу лежит `ожидаемое_значение`, то записываем `новое_значение` и возвращаем `true`;
- если нет — ничего не меняем и возвращаем `false`.

Пример с `AtomicInteger`:
```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicDemo {

    private static final AtomicInteger counter = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            for (int i = 0; i < 1000; i++) {
                counter.incrementAndGet(); // атомарное ++
            }
        };

        Thread t1 = new Thread(task);
        Thread t2 = new Thread(task);

        t1.start(); 
        t2.start();

        t1.join(); 
        t2.join();

        System.out.println("Result: " + counter.get()); // всегда 2000
    }
}
```
Метод `incrementAndGet()` внутри использует CAS, а не `synchronized`.

#### CAS vs synchronized

- **CAS**:
  - Не блокирует поток — работает быстрее при низком конфликте;
  - Хорош для простых операций (`++`, `setIfAbsent` и тп);
  - Используется во многих коллекциях (`ConcurrentHashMap`).

- **synchronized**:
  - Блокирует поток — может быть медленнее;
  - Зато подходит для сложных критических секций (несколько полей, логика).

#### Потенциальные минусы CAS

1. **Spinning (ожидание в цикле)**. Если несколько потоков постоянно конфликтуют, CAS будет многократно пробовать обновить значение, тратя CPU.
2. **ABA-проблема**. Значение изменилось `A`→`B`→`A`. CAS не заметит, что что-то произошло. Для этого в Java есть `AtomicStampedReference` (с версией).

## ThreadLocal — память «на поток»

Вместо того, чтобы синхронизировать доступ к переменной между потоками, иногда проще сделать так, чтобы у каждого потока была своя копия этой переменной. Для этого в Java есть класс `ThreadLocal`.

### Пример без ThreadLocal
```java
public class NoThreadLocalDemo {

    private static int counter = 0;

    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            counter++;
            System.out.println(Thread.currentThread().getName() + " -> " + counter);
        };

        Thread t1 = new Thread(task, "T1");
        Thread t2 = new Thread(task, "T2");

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```
Результат может быть непредсказуемым: два потока меняют один и тот же `counter`.

### Пример с ThreadLocal

```java
public class ThreadLocalDemo {
    private static final ThreadLocal<Integer> counter =
            ThreadLocal.withInitial(() -> 0);

    public static void main(String[] args) throws InterruptedException {
        Runnable task = () -> {
            int value = counter.get();
            
            counter.set(value + 1);
            
            System.out.println(
                Thread.currentThread().getName() + " -> " + counter.get()
            );
        };

        Thread t1 = new Thread(task, "T1");
        Thread t2 = new Thread(task, "T2");

        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}
```
Вывод всегда будет:
```text
T1 -> 1
T2 -> 1
```

У каждого потока — **своя копия** `counter`, никаких гонок.

Где это применяют:
- Хранение контекста (например, `SecurityContext`, `UserSession`, `Transaction`).
- Форматирование дат (`DateFormat` раньше был не потокобезопасен).
- Генерация `traceId`/`logId` для логов.

### Важный нюанс: утечки памяти

`ThreadLocal` хранит данные в карте внутри `Thread`. Если поток живет долго (например, из пула), то и данные остаются. Нужно вызывать `remove()`, чтобы не было утечек.

```java
try {
    counter.set(42);
    // работа с потоком
} finally {
    counter.remove(); // обязательно
}
```
