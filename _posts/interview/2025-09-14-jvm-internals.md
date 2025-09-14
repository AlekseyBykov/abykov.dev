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

Состоит из трех шагов:

1. **Verification** — проверка байткода на корректность и безопасность (работа со стеком, доступ к методам).
2. **Preparation** — выделение памяти под статические поля и присвоение им значений по умолчанию.
3. **Resolution** — замена символических ссылок (например, имя метода) на прямые ссылки в памяти.

### Initialization

На этом шаге выполняются статические блоки инициализации (`static { ... }`) и присваиваются реальные значения static-переменным.
