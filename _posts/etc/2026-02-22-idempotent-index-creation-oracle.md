---
title: "Идемпотентное создание индексов в Oracle: защита от ORA-01408"
date: 2026-02-22
categories:
  - infrastructure
  - oracle
  - database
tags:
  - oracle
  - plsql
  - indexing
  - migrations
  - junit
  - integration-testing
---

Задача идемпотентного создания индексов остается актуальной и сегодня.

Миграции регулярно выполняются повторно — при rollback, re‑deploy, blue‑green‑развертываниях и аварийных перезапусках. Окружения часто живут асимметрично (dev, test, stage, prod), часть изменений может вноситься вручную, а порядок применения релизов различается.

В Oracle до сих пор отсутствует встроенная поддержка:

```sql
CREATE INDEX IF NOT EXISTS ...
```

Поэтому контроль повторного создания индексов по‑прежнему реализуется на уровне прикладной логики.

В миграционных сценариях индексы должны создаваться безопасно и повторяемо. Повторный запуск скрипта не должен приводить к ошибкам — независимо от состояния окружения.

В реальных enterprise‑системах релизные SQL‑скрипты выполняются:

* на нескольких площадках;
* в отдельных экземплярах системы;
* с разной историей ручных доработок;
* повторно при восстановлении из бэкапа;
* повторно при аварийном перезапуске деплоя.

Если создание индекса не является идемпотентным, это приводит к нестабильности релизного процесса.

## Контекст задачи

В типовом релизе обычно требуется добавить один или несколько индексов. На первый взгляд задача тривиальна:

```sql
CREATE INDEX IX_SAMPLE ON SOME_TABLE (COL1, COL2);
```

Однако в реальных окружениях возможны следующие ситуации:

* индекс уже создан ранее вручную DBA;
* индекс существует, но имеет другое имя;
* индекс был добавлен в рамках локальной доработки;
* порядок применения релизов отличался между площадками.

В таких случаях выполнение `CREATE INDEX` приводит к ошибке:

```sql
ORA-01408: such column list already indexed
```

Ошибка означает, что в базе уже существует индекс с тем же набором колонок — даже если его имя отличается.

В результате:

* релизный скрипт прерывается;
* деплой останавливается;
* требуется ручное вмешательство;
* окружения начинают расходиться по состоянию схемы.

Таким образом, задача сводится не к простому созданию индекса, а к обеспечению безопасного и повторяемого поведения миграции.

## Что означает ORA-01408

Ошибка

```sql
ORA-01408: such column list already indexed
```

возникает при попытке создать индекс, если в таблице уже существует другой индекс с тем же набором колонок и тем же порядком.

Важно понимать, что Oracle сравнивает индексы не только по имени. Для базы данных ключевым является именно набор колонок и их позиция внутри индекса.

Например, следующие команды считаются эквивалентными с точки зрения набора колонок:

```sql
CREATE INDEX IX_A ON T (COL1, COL2);
CREATE INDEX IX_B ON T (COL1, COL2);
```

Несмотря на разные имена (`IX_A` и `IX_B`), Oracle воспринимает второй индекс как дубликат первого и генерирует `ORA-01408`.

Проверка только по имени индекса не решает проблему:

```sql
SELECT COUNT(*)
FROM USER_INDEXES
WHERE INDEX_NAME = 'IX_A';
```

Такая проверка отвечает лишь на вопрос, существует ли индекс с конкретным именем. Она не учитывает:

* наличие индекса с тем же набором колонок под другим именем;
* порядок колонок внутри индекса;
* возможные различия между окружениями.

С точки зрения Oracle дубликатом считается индекс, у которого совпадают:

* таблица (`TABLE_NAME`);
* количество колонок;
* значение `COLUMN_POSITION` для каждой колонки;
* имя колонки (`COLUMN_NAME`) в соответствующей позиции.

Именно эта логика лежит в основе ошибки `ORA-01408` и должна быть учтена при построении идемпотентного механизма создания индексов.

## Требования к решению

Для того чтобы создание индекса было безопасным в миграционном сценарии, механизм должен удовлетворять нескольким требованиям.

- **Идемпотентность**. Повторный запуск процедуры или скрипта не должен изменять состояние схемы и не должен приводить к ошибке. Независимо от того, был ли индекс создан ранее, результат выполнения должен быть предсказуемым.
- **Проверка по имени**. Если индекс с заданным именем уже существует, создание не должно выполняться повторно. Это базовый уровень защиты, предотвращающий дублирование по идентификатору.
- **Exact‑проверка по колонкам**. Необходимо определить, существует ли индекс с тем же набором колонок, даже если его имя отличается. Проверка должна быть строгой и не допускать частичных совпадений.
- **Учет порядка колонок**. Индекс `(COL1, COL2)` и индекс `(COL2, COL1)` — разные структуры с точки зрения оптимизатора. Поэтому сравнение должно учитывать позицию каждой колонки (`COLUMN_POSITION`).
- **Безопасность при повторном запуске**. Даже при конкурентном выполнении или частично примененном релизе механизм не должен прерывать деплой. Ошибка `ORA‑01408` должна быть либо предотвращена заранее, либо корректно обработана.

## Архитектура проверки

### 1. Проверка по имени

Первый уровень защиты — проверка существования индекса по имени. Используется представление `USER_INDEXES`:

```sql
select count(*)
from user_indexes
where index_name = upper(:index_name);
```
Если индекс с таким именем уже существует — создание пропускается. Это защищает от повторного запуска скрипта с тем же именем индекса.

### 2. Проверка по колонкам

Проверки только по имени недостаточно. Индекс с тем же набором колонок может уже существовать под другим именем.

Для exact-проверки используются системные представления Oracle (data dictionary):

- `USER_INDEXES` — список индексов таблицы;
- `USER_IND_COLUMNS` — список колонок индекса;
- `COLUMN_POSITION` — порядок колонок в индексе.

Алгоритм:

1. Получить все индексы таблицы из `USER_INDEXES`.
2. Для каждого индекса:
- сравнить количество колонок;
- проверить совпадение колонок по `COLUMN_POSITION`;
- учитывать порядок колонок как значимый.

Если найден индекс с полностью совпадающим набором колонок в том же порядке — создание нового индекса не выполняется.

**Почему не LIKE**

Альтернативный подход — агрегировать список колонок в строку и сравнивать через `LIKE`. Этот способ проще, но менее надежен:

- возможны ложные совпадения;
- возможны ошибки при функциях и выражениях;
- чувствителен к форматированию;
- не гарантирует exact-совпадение структуры.

Поэтому в финальном решении используется проверка через dictionary.

### 3. Создание индекса

Если индекс не существует по имени и не найден exact-дубликат по колонкам, формируется динамический SQL:

```sql
create index <index_name>
on <table_name> (<column_list>)
```

Используется `EXECUTE IMMEDIATE`, так как имя индекса, таблицы и список колонок передаются параметрами.

Дополнительно перехватывается ошибка `ORA-01408`:

```sql
if sqlcode = -1408 then
    -- such column list already indexed
    return;
end if;
```

Это финальный защитный уровень на случай конкурентного создания или нестандартного состояния схемы.

## Финальная процедура

Реализация строится вокруг трех логических блоков:

```sql
if exists(index by name)
    return

if exists(index by exact columns)
    return

create index
```

**Нормализация входных данных**

- приведение имен к UPPER
- формирование строки списка колонок
- проверка на пустой набор

```sql
-- формирование списка колонок
for i in 1 .. index_columnNames.count loop
    ...
end loop;
```

**Проверка существования**

Сначала проверка по имени:
```sql
select count(*)
into v_cnt
from user_indexes
where index_name = upper(indexName);
```

Затем exact-проверка по колонкам через `USER_IND_COLUMNS` и `COLUMN_POSITION`.

Логика сравнения:

- одинаковое количество колонок;
- совпадение имени колонки;
- совпадение позиции.

**Создание индекса**

Если совпадений не найдено:

```sql
execute immediate v_sql;
```

**Полный листинг**

```sql
-- (10g-safe + EXACT check + optional debug)
create or replace procedure createIndexProc (
    indexName           varchar2,
    tableName           varchar2,
    index_columnNames   TYPE_TABLE_OF_VARCHAR2
) is
    ------------------------------------------------------------------
    -- Settings
    ------------------------------------------------------------------
    c_debug constant boolean := true;

    ------------------------------------------------------------------
    -- State
    ------------------------------------------------------------------
    v_cols_csv    varchar2(4000);
    v_cols_count  pls_integer := 0;
    v_sql         varchar2(4000);

    ------------------------------------------------------------------
    -- Debug helper
    ------------------------------------------------------------------
    procedure log(p_msg varchar2) is
    begin
        if c_debug then
            dbms_output.put_line('[createIndexByColumns] ' || p_msg);
        end if;
    end;

    ------------------------------------------------------------------
    -- Build normalized column list: COL1,COL2,...
    -- (No TABLE() usage, relies on collection indexing)
    ------------------------------------------------------------------
    procedure build_columns(
        p_list   out varchar2,
        p_count  out pls_integer
    ) is
        v_list  varchar2(4000);
        v_cnt   pls_integer := 0;
        v_item  varchar2(4000);
    begin
        v_list := null;

        if index_columnNames is null then
            raise_application_error(-20910, 'index_columnNames IS NULL');
        end if;

        log('collection.count=' || index_columnNames.count);

        for i in 1 .. index_columnNames.count loop
            v_item := index_columnNames(i);
            if c_debug then
                log('collection(' || i || ')=' || nvl(v_item, '<null>'));
            end if;

            if v_item is not null then
                if v_cnt > 0 then
                    v_list := v_list || ',';
                end if;

                v_list := v_list || upper(trim(v_item));
                v_cnt  := v_cnt + 1;
            end if;
        end loop;

        p_list  := v_list;
        p_count := v_cnt;
    end;

    ------------------------------------------------------------------
    -- Name check: index exists by name?
    ------------------------------------------------------------------
    function exists_by_name(p_index varchar2) return boolean is
        v_cnt number;
    begin
        select count(*)
          into v_cnt
          from user_indexes
         where index_name = upper(p_index);

        log('exists_by_name cnt=' || v_cnt);
        return v_cnt > 0;
    end;

    ------------------------------------------------------------------
    -- Exact check: any index on table has EXACT same columns
    -- (same count + same column_name at same column_position)
    ------------------------------------------------------------------
    function exists_by_exact_columns(
        p_table     varchar2,
        p_col_count pls_integer
    ) return boolean is
        v_idx_col_cnt number;
        v_match_cnt   pls_integer;
        v_cnt         number;
    begin
        for r in (
            select index_name
              from user_indexes
             where table_name = upper(p_table)
        ) loop

            select count(*)
              into v_idx_col_cnt
              from user_ind_columns
             where index_name = r.index_name;

            if v_idx_col_cnt = p_col_count then
                v_match_cnt := 0;

                for i in 1 .. p_col_count loop
                    select count(*)
                      into v_cnt
                      from user_ind_columns
                     where index_name      = r.index_name
                       and column_position = i
                       and column_name     = upper(trim(index_columnNames(i)));

                    if v_cnt = 1 then
                        v_match_cnt := v_match_cnt + 1;
                    end if;
                end loop;

                if v_match_cnt = p_col_count then
                    log('exists_by_exact_columns found=' || r.index_name);
                    return true;
                end if;
            end if;

        end loop;

        return false;
    end;

    ------------------------------------------------------------------
    -- Build DDL
    ------------------------------------------------------------------
    function ddl_create_index(
        p_index varchar2,
        p_table varchar2,
        p_cols  varchar2
    ) return varchar2 is
    begin
        return 'create index ' || upper(p_index)
            || ' on ' || upper(p_table)
            || ' (' || p_cols || ')';
    end;

begin
    log('IN indexName=' || indexName || ' tableName=' || tableName);

    if indexName is null or tableName is null then
        raise_application_error(-20912, 'indexName/tableName is NULL');
    end if;

    build_columns(v_cols_csv, v_cols_count);

    log('columns=' || v_cols_csv);
    log('columns_count=' || v_cols_count);

    if v_cols_count = 0 then
        raise_application_error(-20911, 'Column list resolved to empty');
    end if;

    -- 1) by name
    if exists_by_name(indexName) then
        log('skip: index already exists by name');
        return;
    end if;

    -- 2) exact by columns
    if exists_by_exact_columns(tableName, v_cols_count) then
        log('skip: index already exists by exact columns');
        return;
    end if;

    -- 3) create
    v_sql := ddl_create_index(indexName, tableName, v_cols_csv);
    log('DDL=' || v_sql);

    begin
        execute immediate v_sql;
        log('created');
    exception
        when others then
            -- last line of defense for concurrent DDL
            if sqlcode = -1408 then
                log('ORA-01408 caught, skipping');
                return;
            end if;

            log('ERROR=' || sqlerrm);
            raise;
    end;

end;
/
```

Дополнительно перехватывается `ORA-01408` как защитный уровень от конкурентного выполнения.

## Интеграционное тестирование через JUnit

DDL и системный словарь Oracle плохо тестируются "в вакууме": поведение зависит от текущего состояния схемы, а результат проверки читается из `USER_INDEXES` / `USER_IND_COLUMNS`. Поэтому вместо unit-тестов используются интеграционные тесты, которые выполняются на реальной Oracle-схеме и создают временные объекты.

Ниже — примеры тестов на JUnit 4. Вызов процедуры сделан через PL/SQL-блок, чтобы не упираться в особенности JDBC-работы с Oracle-коллекциями.

### Создание индекса

Базовый сценарий: индекс отсутствует — процедура должна создать его.

```java
@Test
public void shouldCreateIndexIfNotExists() throws Exception {
    String table = "CIB_TEST_TABLE";
    String index = "CIB_TEST_IDX";

    try (Statement st = connection.createStatement()) {

        cleanup(st, table, index);

        st.execute("""
            CREATE TABLE CIB_TEST_TABLE (
                COL1 VARCHAR2(100),
                COL2 VARCHAR2(100)
            )
        """);

        callProcedure(st, index, table, "COL1", "COL2");

        try (ResultSet rs = st.executeQuery("""
            select count(*)
            from user_indexes
            where index_name = 'CIB_TEST_IDX'
        """)) {
            rs.next();
            Assert.assertEquals(1, rs.getInt(1));
        }

        cleanup(st, table, index);
    }
}

```
### Идемпотентность

Повторный запуск не должен приводить к ошибке и не должен создавать второй индекс.

```java
@Test
public void shouldBeIdempotent() throws Exception {
    String table = "CIB_TEST_TABLE";
    String index = "CIB_TEST_IDX";

    try (Statement st = connection.createStatement()) {

        cleanup(st, table, index);

        st.execute("""
            CREATE TABLE CIB_TEST_TABLE (
                COL1 VARCHAR2(100),
                COL2 VARCHAR2(100)
            )
        """);

        callProcedure(st, index, table, "COL1", "COL2");
        callProcedure(st, index, table, "COL1", "COL2");

        try (ResultSet rs = st.executeQuery("""
            select count(*)
            from user_indexes
            where index_name = 'CIB_TEST_IDX'
        """)) {
            rs.next();
            Assert.assertEquals(1, rs.getInt(1));
        }

        cleanup(st, table, index);
    }
}
```

### Совпадение по колонкам

Если индекс уже существует с тем же набором колонок, но под другим именем, новый индекс создаваться не должен.

```java
@Test
public void shouldSkipIfIndexExistsByColumns() throws Exception {
    String table = "CIB_TEST_TABLE";
    String existing = "CIB_EXISTING_IDX";
    String newIndex = "CIB_NEW_IDX";

    try (Statement st = connection.createStatement()) {

        cleanup(st, table, existing);
        cleanup(st, table, newIndex);

        st.execute("""
            CREATE TABLE CIB_TEST_TABLE (
                COL1 VARCHAR2(100),
                COL2 VARCHAR2(100)
            )
        """);

        st.execute("""
            create index CIB_EXISTING_IDX
            on CIB_TEST_TABLE (COL1, COL2)
        """);

        callProcedure(st, newIndex, table, "COL1", "COL2");

        try (ResultSet rs = st.executeQuery("""
            select count(*)
            from user_indexes
            where index_name = 'CIB_NEW_IDX'
        """)) {
            rs.next();
            Assert.assertEquals(0, rs.getInt(1));
        }

        cleanup(st, table, existing);
        cleanup(st, table, newIndex);
    }
}
```
### Порядок колонок

Индекс (`COL1`, `COL2`) и (`COL2`, `COL1`) — разные структуры. В этом случае новый индекс должен быть создан.

```java
@Test
public void shouldCreateIfColumnOrderDiffers() throws Exception {
    String table = "CIB_TEST_TABLE";
    String existing = "CIB_EXISTING_IDX";
    String newIndex = "CIB_NEW_IDX";

    try (Statement st = connection.createStatement()) {

        cleanup(st, table, existing);
        cleanup(st, table, newIndex);

        st.execute("""
            CREATE TABLE CIB_TEST_TABLE (
                COL1 VARCHAR2(100),
                COL2 VARCHAR2(100)
            )
        """);

        st.execute("""
            create index CIB_EXISTING_IDX
            on CIB_TEST_TABLE (COL1, COL2)
        """);

        callProcedure(st, newIndex, table, "COL2", "COL1");

        try (ResultSet rs = st.executeQuery("""
            select count(*)
            from user_indexes
            where index_name = 'CIB_NEW_IDX'
        """)) {
            rs.next();
            Assert.assertEquals(1, rs.getInt(1));
        }

        cleanup(st, table, existing);
        cleanup(st, table, newIndex);
    }
}
```

### Вспомогательные методы

Для компактности тестов логика вызова процедуры и очистки вынесена в helper-методы.

```java
private void callProcedure(
        Statement st,
        String indexName,
        String tableName,
        String... columns
) throws Exception {

    StringBuilder plsql = new StringBuilder("""
        DECLARE
            cols TYPE_TABLE_OF_VARCHAR2 := TYPE_TABLE_OF_VARCHAR2();
        BEGIN
    """);

    plsql.append("cols.extend(").append(columns.length).append(");");

    for (int i = 0; i < columns.length; i++) {
        plsql.append("""
            cols(%d) := '%s';
        """.formatted(i + 1, columns[i]));
    }

    plsql.append("""
            createIndexProc('%s', '%s', cols);
        END;
    """.formatted(indexName, tableName));

    st.execute(plsql.toString());
}

private void cleanup(Statement st, String table, String index) {
    try { 
        st.execute("drop index " + index); 
    } catch (Exception ignored) {}
    
    try { 
        st.execute("drop table " + table + " purge"); 
    } catch (Exception ignored) {}
}
```
