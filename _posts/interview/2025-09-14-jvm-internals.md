---
layout: post
title: "JVM Internals: Полный разбор с примерами"
date: 2025-09-14
categories: [java, java-core]
tags: [jvm, internals, classloader, jit, gc]
---

**Java Virtual Machine (JVM)** — это ядро платформы Java. Когда мы запускаем программу, JVM берет на себя задачу загрузки классов, управления памятью, интерпретации или компиляции байткода, а также обеспечивает безопасность и кроссплатформенность. Чтобы понимать, как работает Java «под капотом», нужно заглянуть в устройство JVM и разобраться в ее подсистемах.

В прошлый раз, в статье ["Java под капотом: от исходников к байткоду и оптимизациям"](/posts/java-platform-bytecode-jvm/) мы разобрались с JDK, JRE и JVM в целом, теперь посмотрим немного глубже во внутренности JVM.

## Спецификация и реализации JVM

Важно помнить: *JVM — это спецификация, а не конкретная программа*.

В спецификации описано, как должны загружаться классы, какие есть области памяти, как исполняется байткод. На основе этой спецификации разные компании создавали свои реализации виртуальной машины:

- **HotSpot JVM** (Oracle, **OpenJDK**) — самая популярная, входит в JDK и JRE по умолчанию.
- **OpenJ9** (Eclipse Foundation, ранее IBM J9) — экономнее по памяти, подходит для облаков и контейнеров.
- **GraalVM** — современная JVM, которая добавляет новый JIT-компилятор Graal и поддержку AOT-компиляции в нативный бинарь.
- Другие (**Avian**, **KVM**) — нишевые и менее популярные.

Независимо от реализации, структура JVM всегда включает три больших подсистемы:

- **ClassLoader Subsystem** — загрузка классов.
- **Runtime Data Areas** — управление памятью.
- **Execution Engine** — выполнение байткода (интерпретация, JIT).

## ClassLoader Subsystem

JVM не загружает все классы сразу. Классы подгружаются динамически, в момент первого обращения. Этим занимается **ClassLoader Subsystem**, который работает в три фазы:

1. **Loading** — поиск и загрузка байткода.
2. **Linking** — подготовка класса к использованию (verification, preparation, resolution).
3. **Initialization** — выполнение статических инициализаторов и присвоение значений static-полям.

### Loading

ClassLoader ищет `.class`-файл в файловой системе, JAR-архиве или по сети, считывает байткод и передает его JVM. В Java работает иерархия загрузчиков:

- **Bootstrap ClassLoader** — встроен в JVM, загружает базовые классы (`java.lang.*`, `java.util.*`).
- **Platform (Extension) ClassLoader** — загружает стандартные модули и расширения (JDBC, XML).
- **Application ClassLoader** — загружает классы приложения с classpath.

По умолчанию действует **parent delegation model** (*делегирование вверх*): ClassLoader сначала спрашивает у родителя, и только если тот не нашел — ищет сам. Это защищает от подмены базовых классов.

#### Parent delegation model

```text
Запрос: "com.example.Main"
         |
   [Application CL] — ищет в classpath
         ↑
   [Platform CL] — ищет среди модулей (JDBC, XML и т.п.)
         ↑
   [Bootstrap CL] — ищет среди core-классов (java.lang, java.util)
```

1. Application ClassLoader (AppCL) получает запрос на загрузку класса, например `com.example.Main`. Сначала он спрашивает у родителя - Platform CL.
2. Platform ClassLoader (PlatCL) получает этот запрос и передает его дальше — Bootstrap CL.
3. Bootstrap ClassLoader (BootCL) получает запрос и ищет класс:
- если это базовые пакеты (`java.lang.*`, `java.util.*`, `java.sql.*`, `java.xml.*` и пр.), он его загрузит;
- если не нашел — возвращает «не знаю».
4. Если BootCL не нашел — запрос идет обратно в PlatCL:
- PlatCL ищет среди стандартных модулей Java, например JDBC-драйверы, XML-парсеры и тп;
- если PlatCL не нашел — возвращает «не знаю».
5. Если и PlatCL не нашел — очередь за AppCL:
- AppCL ищет в classpath приложения (JAR-ы, директории), если там есть `com.example.Main.class` — загрузит.
- Если AppCL не нашел — значит, не нашел ни один загрузчик, выбрасывается `ClassNotFoundException`.

> ⚠️ То есть порядок всегда один: сначала родительский зарузчик, потом сам, т.е. делегирование вверх. Но что именно загрузит каждый — зависит от 
> того, чей это класс: базовый (`java.lang.String`), модульный (`javax.sql.DataSource`), или наш (`com.example.Main`).

**Краткая таблица «кто за что отвечает»**

| Загрузчик              | Что загружает                                                         |
| ---------------------- | --------------------------------------------------------------------- |
| **Bootstrap CL**       | Базовые классы JDK (`java.lang`, `java.util`, `java.sql`, `java.xml`) |
| **Platform CL**        | Расширения платформы (`java.sql`, `java.desktop`, JDBC, XML и др.)    |
| **Application CL**     | Классы приложения из `classpath` (JAR-ы и директории)                 |
| **Custom ClassLoader** | Все, что мы захотим загрузить сами (плагины, модули, сети)            |


Рассмотрим на примере.

```java
public class ClassLoaderDemo {

    public static void main(String[] args) {
        // Базовый класс из java.lang
        System.out.println("String -> " + String.class.getClassLoader());

        // Класс из стандартной библиотеки (коллекции)
        System.out.println("ArrayList -> " 
               + java.util.ArrayList.class.getClassLoader());

        // JDBC API (часто грузится Platform CL)
        System.out.println("javax.sql.DataSource -> " 
               + javax.sql.DataSource.class.getClassLoader());

        // Наш собственный класс
        System.out.println("ClassLoaderDemo -> " 
               + ClassLoaderDemo.class.getClassLoader());
    }
}
```
В результате увидим:
```bash
String -> null
ArrayList -> null
javax.sql.DataSource -> jdk.internal.loader.ClassLoaders$PlatformClassLoader@2f92e0f4
ClassLoaderDemo -> jdk.internal.loader.ClassLoaders$AppClassLoader@74a14482

```
`null` - это не ошибка, а значит, что класс загружен Bootstrap ClassLoader (например, `String`, `ArrayList`). PlatformClassLoader - это Platform CL, отвечает за «средние» модули. AppClassLoader - загрузчик нашего приложения, берет классы из classpath.

Так мы наглядно видим:

- Базовые классы JDK (`java.*`) сидят в Bootstrap,
- Модули уровня платформы — в Platform,
- Все наше приложение — в AppClassLoader.

#### Пример с кастомным ClassLoader

По умолчанию в Java работает делегирование вверх: если AppClassLoader не находит класс, он передает запрос родителю (Platform → Bootstrap).
Но мы можем вмешаться в этот механизм и написать свой ClassLoader.

Это нужно, если:
- Необходимо подгружать классы «на лету» (например, из базы данных или сети);
- Реализуется плагинная система (у каждого плагина — свои зависимости);
- Требуется изолировать версии библиотек внутри одного процесса.

Схема с кастомным загрузчиком может выглядеть так:
```text
       [Bootstrap ClassLoader]
                ↑
       [Platform ClassLoader]
                ↑
   [Application (App) ClassLoader]
                ↑
      [Custom PluginClassLoader]
                ↑
           [User Plugin Classes]
```
В этом случае обычные классы идут по стандартной цепочке, а плагины загружаются отдельно, через наш загрузчик.

Рассмотрим пример.

```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public class CustomClassLoader extends ClassLoader {

    private final Path classesDir;

    public CustomClassLoader(Path classesDir) {
        super(CustomClassLoader.class.getClassLoader()); // parent = AppClassLoader
        this.classesDir = classesDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            String fileName = name.replace('.', '/') + ".class";
            Path classFile = classesDir.resolve(fileName);

            byte[] classBytes = Files.readAllBytes(classFile);

            return defineClass(name, classBytes, 0, classBytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException("Не удалось загрузить " + name, e);
        }
    }

    public static void main(String[] args) throws Exception {
        Path path = Path.of("custom-classes");
        CustomClassLoader loader = new CustomClassLoader(path);

        Class<?> helloClass = loader.loadClass("demo.Hello");

        System.out.println("Класс: " + helloClass);
        System.out.println("Загрузчик: " + helloClass.getClassLoader());
    }
}

```
Если в папке `custom-classes` лежит `demo.Hello.class`, получим такой вывод:
```bash
Класс: class demo.Hello
Загрузчик: CustomClassLoader@5e91993f
```

Таким образом, цепочку загрузки классов можно не только использовать, но и расширять. Это один из мощных механизмов JVM, который активно применяют контейнеры приложений (Tomcat, OSGi, Spring Boot).

#### Что будет, если нарушить делегирование

В Java классы считаются одинаковыми **только если они загружены одним и тем же ClassLoader-ом**. Если два разных загрузчика загрузят один и тот же `.class` — это будут два разных класса для JVM.

Самый показательный пример — если мы попытаемся загрузить стандартный класс (`java.lang.String`) через кастомный загрузчик:
```java
public class StringLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        if (name.equals("java.lang.String")) {
            throw new ClassNotFoundException("Нельзя грузить String вручную!");
        }
        return super.findClass(name);
    }

    public static void main(String[] args) throws Exception {
        StringLoader loader = new StringLoader();
        // Попробуем явно загрузить String
        loader.loadClass("java.lang.String");
    }
}

```
В результате увидим:
```java
Exception in thread "main" java.lang.ClassNotFoundException: java.lang.String
```

JVM защищает базовые классы: они всегда грузятся **только Bootstrap ClassLoader-ом**. Если попытаться обойти правило — возникнет ошибка.


#### А если загрузить «обычный» класс повторно?

Рассмотрим пример:

```java
Class<?> c1 = ClassLoader.getSystemClassLoader().loadClass("demo.Hello");
CustomClassLoader custom = new CustomClassLoader(Path.of("custom-classes"));
Class<?> c2 = custom.loadClass("demo.Hello");

System.out.println(c1 == c2); // false!
```

Хотя имя класса одинаковое (`demo.Hello`), для JVM это два разных класса: `c1` загружен AppClassLoader-ом, `c2` загружен CustomClassLoader-ом.

Попробуем привести объект к «чужому» типу:
```java
Object o = c2.getDeclaredConstructor().newInstance();
demo.Hello h = (demo.Hello) o; // ClassCastException!
```

Увидим ошибку:
```java
ClassCastException: class demo.Hello (loaded by CustomClassLoader)
cannot be cast to class demo.Hello (loaded by AppClassLoader)
```

Таким образом, базовые классы (`java.*`) нельзя перегрузить — они жестко закреплены за Bootstrap, для остальных классов два разных загрузчика - два разных мира типов.

### Linking

Фаза связывания делает загруженный класс готовым к исполнению. Она состоит из трех шагов:

1. **Verification** — проверка байткода на корректность и безопасность (работа со стеком, доступ к методам).
2. **Preparation** — выделение памяти под статические поля и присвоение им значений по умолчанию.
3. **Resolution** — замена символических ссылок (например, имя метода) на прямые ссылки в памяти.

Рассмотрим подробно каждый шаг.

#### Verification (верификация)
JVM проверяет байткод, чтобы убедиться, что он корректен и безопасен:

- Стековые операции сбалансированы (push/pop совпадают);
- Вызовы методов соответствуют сигнатурам;
- Нет доступа к приватным полям/методам из другого класса;
- Нет некорректных переходов (например, в середину инструкции).

Если проверка не пройдена — выбрасывается `java.lang.VerifyError`.

#### Preparation (подготовка)

На этом шаге выделяется память под static-поля класса и они инициализируются значениями по умолчанию (`0`, `null`, `false`).

Рассмотрим пример.
```java
class Demo {
    static int a = 42;
    static String s = "hi";
}
```
На рассматриваемом этапе Preparation `a = 0`, `s = null`. Значения из инициализаторов (`42`, `"hi"`) будут присвоены позже — на этапе **Initialization**.

#### Resolution (разрешение ссылок)

Символические ссылки (например, имя метода `"foo"`) заменяются на прямые ссылки в памяти: адреса методов, полей, классов. Это ускоряет выполнение.

Например, при вызове `obj.toString()` JVM сначала ищет метод в классе `Object`, затем кэширует эту ссылку. Следующие вызовы идут напрямую.

### Initialization

Финальная стадия — выполнение статических инициализаторов и присвоение реальных значений static-полям.

Пример:
```java
class InitDemo {
    static int a = 42;

    static {
        System.out.println("Static block runs!");
        a = 100;
    }
}
```
Порядок инициализации:

1. На этапе Preparation: `a = 0`.
2. На этапе Initialization: сначала `a = 42`, затем выполняется `static { ... }` и `a = 100`.

При первом обращении к `InitDemo.a` JVM запустит инициализацию класса. Если мы вызовем:
```java
System.out.println(InitDemo.a);
```
Вывод будет:
```bash
Static block runs!
100
```

Инициализация происходит лениво — только при первом обращении к классу или его статическим членам. Если класс не используется — он может так и остаться неинициализированным. Исключения в `static {}` блоке завершают инициализацию с ошибкой `ExceptionInInitializerError`.

Таким образом, **Loading** → **Linking** → **Initialization** — это три обязательных шага перед тем, как класс реально начнет работать.

## Runtime Data Areas

JVM управляет памятью, разделяя ее на несколько областей, каждая из которых выполняет свою роль. Эти области называются **Runtime Data Areas** и создаются при запуске виртуальной машины. Часть из них общая для всех потоков, часть — выделяется отдельно для каждого потока.

Рассмотрим общую схему.

```
+-------------------+   (одна на процесс JVM)
|     Heap          |  ← объекты, управляет GC
+-------------------+
|  Method Area      |  ← метаданные классов, константный пул
|  (Metaspace в JDK8+)
+-------------------+
            ↑
-----------------------------------------------
| Каждому потоку JVM при создании выделяется: |
-----------------------------------------------
|  Java Stack       |  ← фреймы вызовов, локальные переменные
|  PC Register      |  ← текущая инструкция
|  Native Method St.|  ← вызовы JNI и нативные методы
-----------------------------------------------

```

### Heap (Куча)

Это общая область памяти для всех потоков, здесь размещаются **объекты** и **массивы**. Управляется **сборщиком мусора** (**GC**).

Делится на поколения:
- **Young Generation**(Eden + два Survivor) — «молодые» объекты, чаще всего быстро умирают.
- **Old Generation** (Tenured) — объекты, пережившие несколько сборок, считаются «старшими».

Пример:
```java
class Person {
    String name;
}

public class Demo {
    public static void main(String[] args) {
        Person p = new Person(); // объект Person создается в куче
    }
}
```

Если куча переполнена (слишком много объектов или утечка памяти), возникает ошибка:
```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space

```

### Java Stack

У **каждого потока** свой стек. При вызове метода создается **stack frame** (фрейм вызова), в котором хранятся:

- **Локальные переменные**:
  - параметры метода;
  - локальные переменные (`int`, `Object` ссылки и тп).
- **Операндный стек**:
  - используется для вычислений: например, чтобы сложить два числа, JVM кладет их на стек и вызывает инструкцию `iadd`.
  - работает по принципу LIFO.
- **Ссылка на constant pool**:
  - каждый класс имеет constant pool (таблица строк, методов, символов).
  - фрейм хранит указатель, чтобы быстро находить ссылки на методы и поля.

Фрейм уничтожается при завершении метода.

Пример:
```java
public class StackDemo {

    public static void recursive(int x) {
        recursive(x + 1); // переполняем стек
    }

    public static void main(String[] args) {
        recursive(1);
    }
}
```

Схема стека потока `main` во время рекурсии:
```text
Thread-1 Stack
+----------------------------+
| Frame: recursive(3)        | ← активный метод
|  - локальные переменные    |
|  - операндный стек         |
|  - ссылка на constant pool |
+----------------------------+
| Frame: recursive(2)        |
+----------------------------+
| Frame: recursive(1)        |
+----------------------------+
| Frame: main()              |
+----------------------------+

```

Какие тут могут быть ошибки:

- `StackOverflowError` — бесконечная рекурсия или слишком глубокий вызов методов.
- `OutOfMemoryError` — если JVM не смогла выделить память под стек нового потока.

Результат выполнения примера выше: 
```java
Exception in thread "main" java.lang.StackOverflowError.
```

### Program Counter (PC Register)

У каждого потока есть свой счетчик команд (Program Counter), который хранит адрес текущей инструкции байткода, выполняемой JVM. Счетчик нужен, чтобы при переключении потоков JVM могла продолжить выполнение с правильного места. Для нативных методов (через JNI) значение PC может быть `undefined`.

### Native Method Stack

Используется для вызовов **нативных методов** (реализованных на C/C++ через JNI).
Содержит стековые фреймы нативного кода.

Ошибки:

- `StackOverflowError` — переполнение нативного стека.
- `OutOfMemoryError` — не удалось выделить память.


Пример:
```java
class NativeDemo {
    static {
        System.loadLibrary("demo"); // загрузка нативной библиотеки
    }
    public native void nativeMethod();
}
```

Тело метода nativeMethod реализуется в C/C++ и выполняется через Native Stack.

### Method Area (Metaspace)

Общая область памяти для всех потоков, где JVM хранит:

- Метаданные классов (имена, поля, методы);
- Константный пул (строки, ссылки на методы и поля);
- Байткод методов;
- Статические переменные.

До Java 8 использовалась PermGen (Permanent Generation), далее ее заменил Metaspace, который размещается в нативной памяти.

Если в систему загружается слишком много классов (например, при динамической генерации), возникает ошибка:
```java
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
```

> ⚠️ Таким образом:
>
> - Heap → объекты.
> - Java Stack → вызовы методов, локальные переменные.
> - PC → текущая инструкция.
> - Native Stack → нативные методы.
> - Method Area / Metaspace → метаданные классов и константы.

### Мини-пример: стек и constant pool

Возьмем простую программу:
```java
public class Calc {

    public static int sum(int a, int b) {
        return a + b;
    }

    public static void main(String[] args) {
        int r = sum(2, 3);
        System.out.println(r);
    }
}
```
Скомпилируем и посмотрим байткод:
```bash
javac Calc.java
javap -c Calc
```

Результат (обрезанный для наглядности):

```java
public static int sum(int, int);
  Code:
     0: iload_0      // загрузить аргумент a в стек
     1: iload_1      // загрузить аргумент b в стек
     2: iadd         // снять два числа со стека, сложить, результат положить в стек
     3: ireturn      // вернуть верхнее значение стека

public static void main(java.lang.String[]);
  Code:
     0: iconst_2       // константа 2 -> стек
     1: iconst_3       // константа 3 -> стек
     2: invokestatic #2  // вызов sum(2,3)
     5: istore_1       // сохранить результат в локальную переменную
     6: getstatic #3   // ссылка на System.out из constant pool
     9: iload_1        // загрузить результат обратно в стек
    10: invokevirtual #4  // вызвать println(int)
    13: return
```

Что здесь видно:

- Аргументы метода (`a`, `b`) и константы (`2`, `3`) кладутся в **операндный стек**.
- Инструкция `iadd` снимает два значения со стека и кладет результат обратно.
- Для доступа к `System.out` JVM обращается в **constant pool** (`getstatic #3`).
- Каждая инструкция работает со стеком, а не с регистрами процессора напрямую.

ASCII-схема работы стека при `sum(2,3)`:
```text
Стек: []
iload_0 (a=2)  -> [2]
iload_1 (b=3)  -> [2, 3]
iadd           -> [5]
ireturn        -> вернуть 5
```

Когда JVM исполняет данный код, данные распределяются так:

```text
                [Runtime Data Areas JVM]

+-----------------------------------------------------+
|                     Heap (общая)                   |
|                                                     |
|  [объект PrintStream out] ← System.out              |
|  [строки "Hello", если бы были литералы]            |
+-----------------------------------------------------+
|        Method Area (Metaspace / Constant Pool)      |
|                                                     |
|  class Calc (метаданные)                            |
|  method sum(bytecode)                               |
|  method main(bytecode)                              |
|  ссылка #3 -> System.out                            |
|  ссылка #4 -> PrintStream.println(int)              |
+-----------------------------------------------------+
|               Java Stack (для main)                 |
|                                                     |
|  Локальные переменные:                              |
|    slot0 = args                                     |
|    slot1 = r                                        |
|                                                     |
|  Операндный стек (динамически меняется):            |
|    iconst_2 → push(2)                               |
|    iconst_3 → push(3)                               |
|    invokestatic sum → pop(2,3), push(5)             |
|    istore_1 → slot1 = 5                             |
|    getstatic #3 → push(System.out)                  |
|    iload_1 → push(5)                                |
|    invokevirtual println → pop(System.out,5)        |
+-----------------------------------------------------+
|   PC Register → указывает на текущую инструкцию     |
+-----------------------------------------------------+
|   Native Method Stack → вызовы println внутри JNI   |
+-----------------------------------------------------+

```

Ключевые моменты:
- Heap хранит объекты (`System.out`, строки, созданные new).
- Method Area (Metaspace) хранит описание класса `Calc`, байткод его методов и constant pool.
- Java Stack у каждого потока свой, там локальные переменные и операндный стек.
- PC Register говорит, какая инструкция байткода сейчас выполняется.
- Native Stack нужен, когда вызов уходит в C/С++ (например, `println` внутри обращается к нативным функциям).

### Пример с объектом в Heap и ссылкой в Stack

Рассмотрим такой пример:

```java
class Person {
    String name;

    Person(String name) {
        this.name = name;
    }
}

public class Demo {
    public static void main(String[] args) {
        Person p = new Person("Person");
    }
}
```

Что произойдет при выполнении `new Person("Person")`:

```text
                [Runtime Data Areas JVM]

+------------------------------------------------------+
|                      Heap                            |
|                                                      |
|  [объект Person#1]                                   |
|    поле name -> ссылка на "Person"                   |
|                                                      |
|  [объект String "Person"]                            |
+------------------------------------------------------+
|         Method Area (Metaspace / Constant Pool)      |
|                                                      |
|  class Person (метаданные)                           |
|    поле name:Ljava/lang/String;                      |
|    конструктор <init>(Ljava/lang/String;)V           |
|                                                      |
|  class Demo (метаданные, main)                       |
+------------------------------------------------------+
|               Java Stack (для main)                  |
|                                                      |
|  Локальные переменные:                               |
|    slot0 = args                                      |
|    slot1 = p → ссылка на Person#1 (в Heap)           |
|                                                      |
|  Операндный стек:                                    |
|    new #2 <Class Person> → push (ref)                |
|    ldc "Person" → push (ref на String)               |
|    invokespecial <init> → связывает name             |
+------------------------------------------------------+
|   PC Register → указывает на текущую инструкцию      |
+------------------------------------------------------+
|   Native Method Stack (если будет вызов JNI)         |
+------------------------------------------------------+
```
Ключевые наблюдения:

- Сам объект `Person` и строка `"Person"` создаются в Heap.
- Переменная `p` в `main` хранится в Java Stack и содержит ссылку на объект в Heap.
- Метаданные класса `Person` и `Demo` лежат в Metaspace.

### Heap: поколения и работа GC

Heap в HotSpot реализации JVM делится на **поколения**. Это нужно для оптимизации сборки мусора:

- Большинство объектов «живут» очень недолго (локальные переменные, временные коллекции);
- Некоторые «живут» долго (кеши, сессии, singletons).

Поэтому Heap делят на области:

```text
+------------------------------------------+
|                 Heap                      |
|                                          |
|   +-------------+   +------------------+ |
|   | Young Gen   |   |   Old Gen        | |
|   | (молодые)   |   | (долгоживущие)   | |
|   |             |   |                  | |
|   | Eden        |   |                  | |
|   | Survivor S0 |   |                  | |
|   | Survivor S1 |   |                  | |
|   +-------------+   +------------------+ |
+------------------------------------------+

```
#### **Young Generation**

- **Eden** — все новые объекты создаются здесь.
- **Survivor Spaces** (**S0** и **S1**) — используются для «выживших» после сборки мусора.

Механизм:

- При заполнении Eden запускается **Minor GC**.
- Объекты, которые все еще «живы», переносятся в Survivor.
- Если объект «пережил» несколько сборок, он «повышается» (promoted) в Old Gen.

В следующем примере почти все массивы уйдут в Eden и быстро соберутся GC, не доходя до Old Gen:

```java
public class YoungGenDemo {

    public static void main(String[] args) {
        for (int i = 0; i < 100_000; i++) {
            byte[] data = new byte[1024]; // создаем временные объекты
        }
    }
}
```

#### **Old Generation (Tenured)**

Здесь лежат долгоживущие объекты. Когда объект «переживает» несколько Minor GC, он перемещается в Old Gen. Сборка в Old Gen называется Major GC или Full GC — она тяжелее и может остановить все потоки (Stop-The-World).

В этом примере объекты не «умирают» быстро, их будут «повышать» в Old Gen:
```java
import java.util.*;

public class OldGenDemo {

    public static void main(String[] args) {        
        List<byte[]> cache = new ArrayList<>();
        for (int i = 0; i < 100_000; i++) {
            byte[] data = new byte[1024];
            cache.add(data); // объект сохраняется → живет долго
        }
    }
}
```
#### **Как работает GC**

- **Minor GC**: быстро очищает Eden и переносит выживших в Survivor.
- **Promotion**: после N сборок (порог зависит от JVM-флагов), объект перемещается в Old Gen.
- **Major/Full GC**: освобождает Old Gen (и иногда весь Heap).

Схематичный пример:

```text
Eden → Minor GC → выжившие → S0
S0 → Minor GC → выжившие → S1
после нескольких циклов → Old Gen
```
Какие здесь могут быть ошибки:

- `OutOfMemoryError: Java heap space` — если Heap полностью заполнен и GC не может освободить место.
- `GC overhead limit exceeded` — если JVM тратит слишком много времени на GC, но памяти все равно мало.

## Execution Engine

Когда классы загружены и подготовлены, их байткод попадает в **Execution Engine** — подсистему JVM, которая выполняет инструкции. В ней есть два ключевых механизма:

- **Интерпретатор** — выполняет байткод построчно.
- **JIT-компилятор (Just-In-Time)** — переводит «горячий» код в машинный.

### Интерпретация

На старте программа выполняется интерпретатором. Каждая инструкция байткода читается, проверяется и исполняется:

```java
public static int sum(int a, int b) {
    return a + b;
}

```
Байткод для метода `sum`:
```text
0: iload_0    // загрузить a
1: iload_1    // загрузить b
2: iadd       // сложить
3: ireturn    // вернуть результат

```
Интерпретатор обрабатывает это пошагово:
```text
push a → push b → iadd → ireturn

```
Минус — медленно, так как каждая инструкция проходит цикл разбора.

### JIT-компиляция

Чтобы ускорить выполнение, JVM включает JIT-компилятор:

1. Код стартует в интерпретаторе.
2. JVM собирает статистику (какие методы вызываются чаще всего).
3. Когда метод «прогревается» (становится hot spot), JIT компилирует его в машинный код.
4. Дальше этот код выполняется напрямую процессором — намного быстрее.

Схематично можно представить следующим образом:
```text
Интерпретация (медленно):
байткод → [интерпретатор] → результат

JIT (после прогрева):
байткод → [JIT компилятор] → машинный код → [CPU напрямую]
```

### Benchmark: прогрев JVM

Пример теста с использованием JMH:

```java
@Benchmark
public int testSum() {
    return sum(2, 3);
}
```

Результат в JMH:
```text
# Warmup Iteration   1: 200 ns/op
# Warmup Iteration   2: 150 ns/op
# Warmup Iteration   3: 50 ns/op
Iteration   1: 20 ns/op
Iteration   2: 18 ns/op
Iteration   3: 18 ns/op
```

Видно, что первые итерации медленные (интерпретатор), потом JIT скомпилировал код и скорость выросла в разы.

## Ошибки и граничные случаи

JVM строго следует спецификации, и при нарушении правил выбрасывает ошибки на разных стадиях: при загрузке, связывании, инициализации, выполнении или в работе с памятью. Рассмотрим основные типы ошибок.

### Ошибки загрузки классов

#### ClassNotFoundException 

Данное исключение возникает, когда класс не найден в момент явной загрузки (например, `Class.forName("Foo")`).

Пример:
```java
public class LoadDemo {

    public static void main(String[] args) throws Exception {
        Class.forName("demo.Missing"); // пытаемся загрузить отсутствующий класс
    }
}
```

Результат:
```java
Exception in thread "main" java.lang.ClassNotFoundException: demo.Missing

```

ASCII-схема:
```text
Application ClassLoader
   |
   +-- ищет demo.Missing → не находит → ClassNotFoundException

```

#### NoClassDefFoundError

Возникает, если класс был доступен во время компиляции, но отсутствует во время выполнения.

Пример:
```java
public class Main {
    public static void main(String[] args) {
        Helper.say(); // класс Helper скомпилирован, но удален из classpath
    }
}
```

Результат:
```java
Exception in thread "main" java.lang.NoClassDefFoundError: Helper
```

### Ошибки байткода

#### VerifyError

JVM проверяет байткод на корректность во время Linking → Verification. Если байткод нарушает правила (например, неверные операции со стеком), будет выброшена ошибка.

Пример (искусственно испорченный байткод):
```bash
javac Demo.java
# редактируем class-файл hex-редактором или используем ASM, чтобы сломать сигнатуры
java Demo
```

Результат:
```bash
Exception in thread "main" java.lang.VerifyError: (class: Demo, method: main ...)
```

### Ошибки памяти

#### StackOverflowError

Переполнение стека метода (рекурсия без выхода). Например:
```java
public class StackOverflowDemo {

    static void recurse() {
        recurse();
    }
    public static void main(String[] args) {
        recurse();
    }
}
```
Результат:
```bash
Exception in thread "main" java.lang.StackOverflowError
```

ASCII-схема:
```text
Thread-1 Stack
+-------------------+
| recurse() frame   | ← переполняется
| recurse() frame   |
| recurse() frame   |
| ...               |
```

#### OutOfMemoryError: Java heap space

Если в куче недостаточно памяти для размещения объекта. Например:
```java
public class HeapOOM {
    public static void main(String[] args) {
        int[] big = new int[Integer.MAX_VALUE]; // слишком большой массив
    }
}
```
Результат:
```bash
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

#### OutOfMemoryError: Metaspace

Слишком много загруженных классов (динамическая генерация). Пример:
```java
import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;

public class MetaspaceOOM {
    public static void main(String[] args) {
        while (true) {
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(Object.class);
            enhancer.setUseCache(false);
            enhancer.setCallback((MethodInterceptor) 
               (o, m, a, proxy) -> proxy.invokeSuper(o, a));
            enhancer.create(); // генерирует новый класс
        }
    }
}
```
Результат:
```bash
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
```
#### OutOfMemoryError: unable to create new native thread

Когда ОС больше не может выделить поток:

```java
public class ThreadOOM {
    public static void main(String[] args) {
        while (true) {
            new Thread(() -> {
                try { Thread.sleep(1000000); } catch (InterruptedException e) {}
            }).start();
        }
    }
}
```
Результат:
```bash
Exception in thread "main" java.lang.OutOfMemoryError: 
    unable to create new native thread
```

## Инструменты

Теория про ClassLoader, память и JIT важна, но еще полезнее уметь «заглянуть внутрь» JVM в реальной программе. Для этого у нас есть встроенные инструменты и утилиты.

### javap — дизассемблер байткода

Утилита javap позволяет увидеть, во что компилятор Java превратил исходный код.

Пример:

```java
public class Calc {
    public static int sum(int a, int b) {
        return a + b;
    }

    public static void main(String[] args) {
        System.out.println(sum(2, 3));
    }
}
```

Компиляция и дизассемблирование:
```bash
javac Calc.java
javap -c Calc
```

Результат (обрезанный):
```java
public static int sum(int, int);
  Code:
     0: iload_0
     1: iload_1
     2: iadd
     3: ireturn

public static void main(java.lang.String[]);
  Code:
     0: iconst_2
     1: iconst_3
     2: invokestatic #2 // Method sum:(II)I
     5: getstatic #3    // Field java/lang/System.out:Ljava/io/PrintStream;
     8: invokevirtual #4 // Method java/io/PrintStream.println:(I)V
```
Сразу видим байткод инструкций (`iload_0`, `iadd`, `ireturn`), ссылки в constant pool (`#2`, `#3`, `#4`).

### jcmd — универсальный инструмент диагностики

`jcmd` умеет многое: от информации о JVM до создания heap dump. Пример использования:

```bash
jcmd <pid> VM.flags          # Параметры запуска JVM
jcmd <pid> Thread.print      # Снимок потоков (thread dump)
jcmd <pid> GC.heap_info      # Информация о куче
jcmd <pid> GC.run            # Запустить сборку мусора
jcmd <pid> GC.heap_dump file=heap.hprof   # Снять heap dump
```

Здесь `<pid>` - это PID процесса, найденный с помощью `jcmd`.

### jmap — работа с кучей

Утилита `jmap` показывает состояние памяти и позволяет снять дамп:

```bash
jmap -heap <pid>     # информация о heap (размеры, GC)
jmap -histo <pid>    # топ объектов по количеству/размеру
jmap -dump:live,format=b,file=heap.hprof <pid>  # heap dump
```

Файл `.hprof` потом можно открыть в VisualVM, Eclipse MAT или YourKit.

### jstack — анализ потоков

Чтобы понять, что делают потоки внутри JVM, используется `jstack`:

```bash
jstack <pid> > threaddump.txt
```

В дампе потоков можно увидеть:

- Состояние потоков (`RUNNABLE`, `WAITING`, `BLOCKED`);
- Deadlock (JVM его сама помечает в конце дампа);
- Stack trace методов.

### jconsole / VisualVM

**jconsole** — простая GUI-утилита, входит в JDK. Позволяет следить за heap, потоками, GC в реальном времени. **VisualVM** — более мощный инструмент: heap dump, профилирование, плагины (например, анализ GC, CPU sampler).

Запуск VisualVM:

```bash
jvisualvm
```

### Современные профилировщики

- **Java Flight Recorder (JFR)** — встроенный в JVM профилировщик. Работает с низкими overhead, подходит для продакшена.
- **Async-profiler** — внешний инструмент, умеет строить flame graph.
- **YourKit**, **JProfiler** — коммерческие продукты с удобным GUI.

Пример запуска JFR:
```bash
jcmd <pid> JFR.start name=myrecord settings=profile filename=recording.jfr
# ... дать поработать приложению ...
jcmd <pid> JFR.stop name=myrecord
```
Файл `recording.jfr` можно открыть в **Java Mission Control**.

## Примеры

Полный код примеров и инструкции доступны в [pets.diagnostic-lab](https://github.com/AlekseyBykov/pets.diagnostic-lab) на GitHub:

### Memory Leak

Чтобы наглядно показать, как возникает `OutOfMemoryError` в результате утечки памяти, рассмотрим простой эксперимент.
```java
public class MemoryLeakSimulator {

    public static void run() {
        List<byte[]> leakyList = new ArrayList<>();
        while (true) {
            leakyList.add(new byte[1024 * 1024]); // 1 MB
            try {
                Thread.sleep(1000);
            } catch (InterruptedException ignored) {}
        }
    }
}
```
Запустим этот код с жестким ограничением по памяти (`-Xms64m -Xmx128m`) и включенным дампом кучи (создастся файл дампа памяти `heapdump.hprof`):
```bash
java -Xms64m -Xmx128m \
     -XX:+HeapDumpOnOutOfMemoryError \
     -XX:HeapDumpPath=./heapdump.hprof \
     -cp target/diagnostic-lab-1.0-SNAPSHOT.jar \
     dev.abykov.pets.diagnostic_lab.MainApp memory
```

Мониторинг через JConsole показывает график, где used heap приближается к максимальному (`-Xmx128m`), без значительных спусков:

![jconsole memory leak](/assets/img/javacore/jconsole-mleak-demo-1.png){: .shadow .rounded }

Утечка памяти может проявляться не сразу, особенно на больших серверах, когда heap достаточно велик: график роста может быть незаметен. В продакшене такие утечки приводят к постепенному снижению производительности, increased GC overhead, и в конце — к OutOfMemory.

Что видно на скрине Overview:

![jconsole memory leak](/assets/img/javacore/jconsole-mleak-demo-2.png){: .shadow .rounded }

1. **Heap Memory Usage (левый верхний график)**
- Линейный рост памяти от ~30 МБ до ~130 МБ.
- Нет характерных пилообразных спадов после GC - сборщик мусора не может освободить память.
- Это ключевой симптом утечки.

2. **Threads (правый верхний график)**
- Всего было 15 потоков, и они оставались стабильными.
- Резкое падение до нуля означает, что JVM аварийно завершилась (из-за OOM).

3. **Classes (левый нижний график)**
- Загружено ~2400 классов, и к моменту завершения число стабильное.
- Нет роста — значит, утечки классов (classloader leak) тут нет.

4. **CPU Usage (правый нижний график)**
- Нагрузка минимальна (0.2–0.4%). 
- Это подтверждает, что проблема именно в удержании объектов в памяти, а не в CPU или активных потоках.

Через некоторое время выбрасывается `java.lang.OutOfMemoryError: Java heap space`:

![idea memory leak](/assets/img/javacore/mleak-demo-2.png){: .shadow .rounded }

Чтобы подтвердить источник утечки, откроем дамп (`heapdump.hprof`) в Eclipse MAT. В *Dominator Tree* сразу видно, что почти вся память удерживается через `main`-поток. Внутри него находится коллекция `ArrayList`, а ее `elementData` хранит массивы `byte[]`, каждый по ~1 МБ:

![eclipse mat dominator tree](/assets/img/javacore/eclipse-mat-mleak-demo-1.png){: .shadow .rounded }

Таким образом, GC не может освободить память: список остается достижимым из `main`, и все вложенные `byte[]` удерживаются, пока приложение не упадет с `OutOfMemoryError`.

Eclipse MAT также генерирует отчет *Leak Suspects Report*. В нем видно, что поток `main` удерживает почти всю память (95.6%), а корень утечки находится в `MemoryLeakSimulator.run()` на строке 13:

![mat leak suspect report](/assets/img/javacore/eclipse-mat-mleak-demo-2.png){: .shadow .rounded }

Такой отчет часто указывает не только на удерживающий объект, но и на конкретный участок кода, 
что существенно ускоряет поиск утечки в реальных приложениях.


## Заключение

JVM — это не просто «черный ящик», который исполняет Java-код. Это целая экосистема из взаимосвязанных подсистем:

- **ClassLoader Subsystem** отвечает за загрузку классов и изоляцию типов.
- **Runtime Data Areas** управляют памятью: куча, стеки, Metaspace.
- **Execution Engine** интерпретирует байткод и применяет JIT-компиляцию для ускорения.
- **GC и память** обеспечивают автоматическое управление объектами.
- **Инструменты** (`jcmd`, `jmap`, `jstack`, VisualVM, JFR) позволяют наблюдать и диагностировать JVM в реальном времени.

Почему это важно? Понимание работы Heap и JIT позволяет правильно тюнить сборщик мусора и избегать «тормозов» в продакшене. Знание устройства JVM помогает быстрее отлаживать ошибки вроде `OutOfMemoryError`, `StackOverflowError`, `ClassNotFoundException` или `VerifyError`. А понимание того, как работают ClassLoader и Metaspace, открывает дорогу к созданию плагинных систем, модульных приложений и гибкому управлению зависимостями.

Понимание JVM Internals — это не академическая теория, а практический инструмент. Это база, без которой сложно эффективно отлаживать, оптимизировать и проектировать Java-приложения.
