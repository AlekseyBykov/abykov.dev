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

У каждого потока свой стек. При вызове метода создается фрейм вызова (stack frame), в котором хранятся:

- Локальные переменные (включая параметры метода),
- Операндный стек (используется для вычислений),
- Ссылка на constant pool класса.

Фрейм уничтожается при завершении метода.

Какие тут могут быть ошибки:

- `StackOverflowError` — бесконечная рекурсия или слишком глубокий вызов методов.
- `OutOfMemoryError` — если JVM не смогла выделить память под стек нового потока.

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

Результат: `Exception in thread "main" java.lang.StackOverflowError`.

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
