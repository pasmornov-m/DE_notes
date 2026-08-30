# Apache Spark

## Введение

**Apache Spark** — это распределённый движок обработки данных, предназначенный для выполнения вычислений на кластере из нескольких машин.

Главная идея Spark заключается в том, что большой набор данных разбивается на части (**partitions**), после чего эти части обрабатываются параллельно на разных вычислительных узлах.

Spark используется для:

* пакетной обработки данных (batch processing);
* потоковой обработки (stream processing);
* выполнения SQL-запросов;
* ETL/ELT-пайплайнов;
* машинного обучения;
* обработки больших объёмов структурированных и неструктурированных данных.

Spark предоставляет API на **Scala, Java, Python (PySpark) и R**.

При работе с DataFrame код часто выглядит как комбинация SQL и Python:

```python
result_df = (
    employees_df
    .filter(F.col("salary") >= 3000)
    .groupBy("department")
    .agg(F.avg("salary").alias("avg_salary"))
)
```

При этом важно понимать: Spark — это не просто библиотека для работы с таблицами. Это распределённый вычислительный движок, который самостоятельно строит план выполнения и распределяет работу по кластеру.

---

## Источники данных

Spark умеет читать и записывать данные во множество источников.

### Файловые системы и объектные хранилища

Наиболее распространённые варианты:

* HDFS
* S3 и S3-compatible storage
* локальная файловая система
* Azure Blob Storage
* Azure Data Lake Storage (ADLS)
* Google Cloud Storage

### Форматы данных

Spark поддерживает, в частности:

* CSV
* JSON
* Parquet
* ORC
* Avro
* текстовые файлы

На практике для аналитических задач чаще всего используют **Parquet**, поскольку это колонночный формат и Spark может читать только необходимые столбцы и эффективнее применять фильтры.

### Базы данных

Через JDBC и специализированные коннекторы Spark может работать с:

* PostgreSQL
* MySQL
* Microsoft SQL Server
* Greenplum
* Oracle
* ClickHouse
* Hive
* Cassandra
* MongoDB
* Elasticsearch

Например, чтение из PostgreSQL через JDBC:

```python
df = (
    spark.read
    .format("jdbc")
    .option("url", "jdbc:postgresql://localhost:5432/db")
    .option("dbtable", "public.users")
    .option("user", "user")
    .option("password", "password")
    .load()
)
```

### Системы потоковой передачи данных

Spark может получать данные, например, из:

* Apache Kafka
* Amazon Kinesis

Также Spark часто используется совместно с табличными форматами и lakehouse-технологиями:

* Apache Iceberg
* Delta Lake
* Apache Hudi

---

## Spark и Pandas

На уровне API Spark DataFrame и Pandas DataFrame действительно похожи:

```python
# Pandas
df[df["age"] > 20]

# PySpark
df.filter(F.col("age") > 20)
```

Однако принцип работы принципиально различается.

**Pandas** обычно работает в памяти одной машины. Поэтому он подходит для данных, которые помещаются в RAM одного компьютера.

**Spark** распределяет данные и вычисления между несколькими машинами.

Упрощённо:

```text
Pandas
Данные
  ↓
Одна машина
  ↓
CPU + RAM одного компьютера
```

```text
Spark
              ┌── Executor 1
Данные ───────┼── Executor 2
              ├── Executor 3
              └── Executor 4
```

Поэтому Spark применяется тогда, когда данные или вычисления становятся слишком большими для одной машины.

Важно, однако, что Spark не всегда быстрее Pandas. Для небольших объёмов данных запуск распределённого вычисления может оказаться медленнее из-за накладных расходов.

---

## Архитектура Spark

Spark-приложение состоит из нескольких основных компонентов.

### Driver

**Driver** — управляющий процесс Spark-приложения.

Он отвечает за:

* выполнение пользовательского кода;
* построение логического плана;
* формирование DAG вычислений;
* разбиение работы на stages;
* создание tasks;
* координацию выполнения;
* взаимодействие с cluster manager.

Именно в процессе Driver запускается основной код приложения.

Например:

```python
df = spark.read.parquet("data/")
df = df.filter(F.col("age") > 20)
df.groupBy("city").count().show()
```

Эти операции описывают вычисление, а Driver определяет, как оно будет выполняться на кластере.

---

### SparkSession

В современных приложениях основным объектом для работы с Spark является:

```python
spark = SparkSession.builder.getOrCreate()
```

`SparkSession` является точкой входа в Spark SQL, DataFrame API и другие высокоуровневые возможности Spark.

---

### Cluster Manager

**Cluster Manager** отвечает за выделение вычислительных ресурсов приложению.

Основные варианты:

* Spark Standalone
* YARN
* Kubernetes

Cluster Manager предоставляет Spark необходимые ресурсы для запуска executors.

---

### Worker Node

**Worker Node** — вычислительный узел кластера, на котором запускаются процессы `executor`.

Один worker может одновременно размещать несколько executors, если это допускается конфигурацией и доступными ресурсами.

---

### Executor

**Executor** — процесс, выполняющий задачи Spark на worker-узле.

Executor:

* выполняет tasks;
* хранит промежуточные данные;
* может кэшировать данные;
* участвует в shuffle;
* возвращает результаты Driver-у.

Внутри одного executor может одновременно выполняться несколько tasks, если ему выделено несколько CPU cores.

---

### Task

**Task** — минимальная единица вычисления Spark.

В типичном сценарии одна task обрабатывает одну partition.

Например, если stage работает с 200 partitions, Spark обычно создаёт 200 tasks для этого stage.

---

## Как Spark выполняет вычисление

Упрощённая схема выглядит так:

```text
Пользовательский код
        ↓
    Driver
        ↓
Logical Plan
        ↓
Physical Plan
        ↓
      Jobs
        ↓
      Stages
        ↓
      Tasks
        ↓
   Executors
        ↓
     Результат
```

---

## DAG и ленивая модель выполнения

Spark использует **ленивое выполнение (lazy evaluation)**.

Это означает, что вызов transformation сам по себе обычно не запускает вычисления.

Например:

```python
df = spark.read.parquet("data/")
df_filtered = df.filter(F.col("age") > 20)
df_grouped = df_filtered.groupBy("city").count()
```

До этого момента Spark в основном строит план будущих вычислений.

Реальное выполнение начинается, когда вызывается **action**:

```python
df_grouped.show()
```

или:

```python
df_grouped.write.parquet("result/")
```

К основным transformations относятся:

```text
filter
select
withColumn
groupBy
join
repartition
```

К actions:

```text
count
collect
show
write
```

Знание этой модели важно, потому что один вызов action может запустить выполнение большого количества ранее описанных операций.

---

## RDD

Исторически основной абстракцией Spark является **RDD (Resilient Distributed Dataset)** — распределённая неизменяемая коллекция объектов.

RDD обладает следующими свойствами:

* распределён между несколькими узлами;
* состоит из partitions;
* является неизменяемым;
* поддерживает fault tolerance;
* допускает ленивые transformations;
* вычисляется при вызове action.

Примеры transformations:

```python
rdd.filter(...)
rdd.map(...)
rdd.flatMap(...)
```

Примеры actions:

```python
rdd.count()
rdd.collect()
rdd.saveAsTextFile(...)
```

Сегодня для большинства ETL и аналитических задач предпочтительнее использовать **DataFrame API и Spark SQL**, поскольку Spark может дополнительно оптимизировать работу с ними.

---

## DataFrame API

**DataFrame** — высокоуровневая распределённая структура данных с именованными столбцами и схемой.

Например:

```python
df = spark.read.parquet("events/")
```

После этого можно использовать:

```python
df.select("user_id", "event_time")
df.filter(F.col("event_name") == "purchase")
df.groupBy("user_id").count()
```

DataFrame API является декларативным: пользователь описывает, **что нужно получить**, а Spark самостоятельно выбирает способ выполнения.

---

## Spark SQL и Catalyst Optimizer

Spark SQL и DataFrame API используют единый механизм оптимизации запросов.

Основной компонент этой системы — **Catalyst Optimizer**.

Упрощённо обработка запроса выглядит так:

```text
SQL / DataFrame API
        ↓
Unresolved Logical Plan
        ↓
Analysis
        ↓
Logical Optimization
        ↓
Physical Planning
        ↓
Physical Plan
        ↓
Execution
```

### 1. Unresolved Logical Plan

Spark сначала строит логическое представление запроса.

На этом этапе некоторые объекты ещё не разрешены:

* имена таблиц;
* имена столбцов;
* типы данных.

### 2. Analysis

Spark обращается к catalog и разрешает:

* таблицы;
* столбцы;
* типы;
* функции.

Получается разрешённый логический план.

### 3. Logical Optimization

Catalyst применяет правила оптимизации.

Например:

* **predicate pushdown** — фильтрация как можно ближе к источнику данных;
* **projection pruning** — чтение только необходимых столбцов;
* упрощение выражений;
* устранение ненужных операций.

Например:

```python
df.select("id", "name").filter(F.col("age") > 20)
```

может быть преобразовано так, чтобы источник данных не читал ненужные столбцы.

### 4. Physical Planning

Spark выбирает физические операторы:

* конкретный алгоритм JOIN;
* способ чтения;
* сортировки;
* агрегации;
* shuffle.

### 5. Execution

После выбора физического плана Spark формирует набор задач, которые будут выполняться на executors.

---

## Job → Stage → Task

Для понимания производительности Spark необходимо понимать эту иерархию.

```text
Application
   └── Job
        ├── Stage
        │    ├── Task
        │    ├── Task
        │    └── Task
        │
        └── Stage
             ├── Task
             └── Task
```

### Job

**Job** — вычислительная задача, запускаемая action.

Например:

```python
df.write.parquet("result/")
```

может создать Job.

Другой action:

```python
df.count()
```

создаст другой Job.

Одно Spark-приложение может содержать много Job.

---

### Stage

**Stage** — часть Job, внутри которой операции могут выполняться без необходимости перемещения данных между партициями через shuffle.

Основная граница между stages появляется там, где возникает **wide dependency**, то есть shuffle.

Например:

```python
df.filter(...)
  .select(...)
  .groupBy("city")
  .count()
```

`filter` и `select` могут выполняться в рамках одного stage, после чего `groupBy` приводит к shuffle и образуется следующий stage.

---

### Task

Каждый stage выполняется как набор tasks.

Обычно:

```text
1 partition → 1 task
```

Поэтому количество partitions напрямую влияет на количество tasks.

Например:

```text
Stage:
100 partitions
↓
100 tasks
```

---

## Пример Job → Stage → Task

Рассмотрим:

```python
df = spark.read.parquet("data/")

df_filtered = (
    df
    .filter(F.col("age") > 20)
)

df_repartitioned = (
    df_filtered
    .repartition(10)
)

df_grouped = (
    df_repartitioned
    .groupBy("city")
    .count()
)

df_grouped.write.parquet("result/")
```

Здесь:

1. `filter()` — узкая операция;
2. `repartition(10)` — shuffle;
3. `groupBy()` — ещё один shuffle;
4. `write()` — action, запускающий Job.

В зависимости от оптимизации физический план может отличаться от такого упрощённого представления, но концептуально важна именно последовательность границ shuffle.

---

## Partition

**Partition** — логическая часть распределённого набора данных.

Например:

```text
100 GB данных
      ↓
100 partitions
      ↓
примерно 1 GB на partition
```

Partition не обязательно имеет одинаковый размер.

Количество partitions влияет на параллелизм:

```text
слишком мало partitions
→ мало параллельных задач
→ плохое использование CPU

слишком много partitions
→ большое количество мелких tasks
→ дополнительные накладные расходы
```

---

## Как Spark читает данные

Предположим, Spark читает большой файл.

```text
10 GB данных
      ↓
разбиение на input partitions
      ↓
tasks
      ↓
executors
```

Важно: конкретный размер partition при чтении не является универсально фиксированным значением `128 MB`. Он зависит от:

* формата данных;
* источника;
* размера файлов;
* конфигурации Spark;
* настроек конкретного коннектора;
* особенностей file splitting.

Например, для файловых источников параметр:

```text
spark.sql.files.maxPartitionBytes
```

задаёт целевой максимальный размер input partition.

---

## Пример параллельного выполнения

Предположим:

* 10 GB данных;
* 8 executors;
* каждый executor имеет 2 cores;
* всего 16 доступных cores.

Если сформировалось 80 partitions, Spark сможет одновременно выполнять до 16 tasks, после чего освободившиеся cores будут получать следующие tasks.

```text
80 partitions
      ↓
80 tasks
      ↓
16 tasks одновременно
      ↓
следующие 16
      ↓
...
```

При этом один worker может запускать несколько executors, а конкретное размещение зависит от cluster manager и конфигурации приложения.

---

## Shuffle

**Shuffle** — это перераспределение данных между partitions, которое требуется, когда дальнейшая операция должна работать с данными, сгруппированными определённым образом.

Например:

```python
df.groupBy("user_id").count()
```

Чтобы собрать все строки одного `user_id` вместе, Spark должен переместить данные между partitions.

Упрощённо:

```text
Executor 1 ───┐
Executor 2 ───┼──→ Shuffle ──→ новые partitions
Executor 3 ───┤
Executor 4 ───┘
```

Shuffle является одной из самых дорогих операций в распределённой обработке из-за:

* сетевого обмена;
* сериализации;
* записи и чтения промежуточных данных;
* возможного spill на диск.

---

## Операции, вызывающие shuffle

Типичные примеры:

```python
groupBy(...)
join(...)
distinct()
orderBy(...)
repartition(...)
```

Некоторые операции могут выполняться без shuffle, если между входными и выходными partitions сохраняется необходимая структура данных.

---

## Exchange в плане Spark

В физическом плане Spark shuffle обычно представлен оператором:

```text
Exchange
```

Например:

```text
== Physical Plan ==
SortMergeJoin
:- Exchange hashpartitioning(user_id, 200)
:  +- Scan parquet
+- Exchange hashpartitioning(user_id, 200)
   +- Scan parquet
```

`Exchange` означает, что Spark должен перераспределить данные.

Для быстрого анализа плана полезно использовать:

```python
df.explain()
```

или:

```python
df.explain("formatted")
```

---

## Как уменьшить Shuffle

Основные подходы:

### Фильтровать данные как можно раньше

Вместо:

```python
df.join(large_df, "user_id") \
  .filter(F.col("date") >= "2026-01-01")
```

обычно лучше отфильтровать данные до тяжёлого JOIN, если это возможно.

```python
filtered_df = (
    large_df
    .filter(F.col("date") >= "2026-01-01")
)

df.join(filtered_df, "user_id")
```

Чем меньше данных участвует в shuffle, тем лучше.

### Broadcast Join

Если одна таблица небольшая, её можно разослать на executors:

```python
from pyspark.sql.functions import broadcast

result = large_df.join(
    broadcast(small_df),
    "user_id"
)
```

Это позволяет избежать обычного shuffle большой таблицы.

### Правильное число partitions

После слишком большого количества partitions появляются многочисленные мелкие задачи.

При слишком маленьком количестве partitions снижается параллелизм.

---

## Repartition и Coalesce

### repartition()

`repartition()` выполняет полноценный shuffle.

```python
df = df.repartition(200)
```

или:

```python
df = df.repartition(200, "user_id")
```

Используется, например, когда требуется:

* увеличить число partitions;
* уменьшить число partitions;
* распределить данные по ключу.

### coalesce()

`coalesce()` обычно используется для уменьшения количества partitions без полного shuffle:

```python
df = df.coalesce(50)
```

Это удобно после сильного уменьшения объёма данных.

Важно: `coalesce()` не является универсальной заменой `repartition()`. При сильном изменении распределения или необходимости равномерно перераспределить данные нужен `repartition()`.

---

## Data Skew

**Data Skew** — это неравномерное распределение данных между partitions.

Например:

```text
Partition 1 → 100 MB
Partition 2 → 120 MB
Partition 3 → 110 MB
Partition 4 → 9 GB
```

Одна task будет работать значительно дольше остальных.

В результате:

* stage будет ждать медленную task;
* возрастает расход памяти;
* возможны OOM;
* увеличивается время shuffle;
* часть executors простаивает.

---

## Как обнаружить Data Skew

Один из главных инструментов — **Spark UI**.

Нужно сравнивать:

* duration tasks;
* Shuffle Read;
* Shuffle Write;
* input size;
* spill.

Если большинство tasks завершается быстро, а одна или несколько работают значительно дольше, это может быть признаком skew.

Распределение ключей можно проверить непосредственно:

```python
(
    df.groupBy("key")
      .count()
      .orderBy(F.desc("count"))
      .show(20)
)
```

Например, если:

```text
key=A → 700 000 000 строк
key=B → 100 000 строк
key=C → 80 000 строк
```

то ключ `A` может создать серьёзный skew.

Особенно опасны значения `NULL`, если большая доля данных имеет одинаковый ключ `NULL`.

---

## Борьба с Data Skew

### 1. Broadcast Join

Если одна сторона JOIN достаточно мала:

```python
large_df.join(
    broadcast(small_df),
    "key"
)
```

Shuffle большой таблицы не требуется.

### 2. Salting

Для тяжёлых ключей можно искусственно разделить один ключ на несколько подстратегий.

Например:

```text
user_123
```

может превратиться в:

```text
user_123_0
user_123_1
user_123_2
...
user_123_99
```

Это называется **salting**.

Но salting усложняет запрос и требует аккуратной реализации, особенно если после JOIN необходимо восстановить исходную семантику.

### 3. Обработка тяжёлых ключей отдельно

Иногда наиболее практичный подход — разделить данные на:

```text
обычные ключи
тяжёлые ключи
```

и использовать для них разные стратегии выполнения.

### 4. Adaptive Query Execution

В Spark есть механизм **AQE (Adaptive Query Execution)**, который позволяет адаптировать физическое выполнение на основе фактических характеристик данных.

Например:

```python
spark.conf.set(
    "spark.sql.adaptive.enabled",
    "true"
)
```

AQE может, в частности, оптимизировать shuffle partitions, влиять на выбор стратегии JOIN и выполнять оптимизации, связанные со skew.

---

## Алгоритмы JOIN в Spark

Spark может выбирать разные стратегии JOIN в зависимости от размера данных, статистики и плана выполнения.

Основные стратегии:

* Broadcast Hash Join
* Sort Merge Join
* Shuffled Hash Join
* Broadcast Nested Loop Join
* Cartesian Product

---

## Broadcast Hash Join

Небольшая таблица передаётся на каждый executor.

```text
small table
    ↓
broadcast
 ↓   ↓   ↓   ↓
E1  E2  E3  E4
```

После этого большая таблица может обрабатываться локально.

Это особенно эффективно для:

```text
fact table + small dimension
```

Например:

```python
large_df.join(
    broadcast(country_df),
    "country_id"
)
```

Главный риск — размер broadcast-таблицы.

Если она слишком большая, executors могут получить нехватку памяти.

---

## Sort Merge Join

Обычно используется для крупных таблиц, когда broadcast невозможен.

Упрощённо:

```text
Table A
  ↓
Shuffle
  ↓
Sort
  ↓
Merge
  ↑
Sort
  ↑
Shuffle
  ↑
Table B
```

Преимущество — хорошая масштабируемость на больших объёмах.

Недостаток — дорогие shuffle и сортировка.

---

## Shuffled Hash Join

Обе таблицы перераспределяются по ключу, после чего для одной из сторон строится hash-структура.

Может быть эффективен при определённых размерах и характеристиках данных, но конкретный выбор алгоритма обычно лучше оставить оптимизатору.

---

## Что влияет на выбор JOIN

Алгоритм JOIN зависит не только от типа SQL JOIN.

Важны:

* размер таблиц;
* наличие статистики;
* распределение данных;
* возможность broadcast;
* условия JOIN;
* количество partitions;
* настройки Spark;
* AQE.

Поэтому утверждение вида:

```text
LEFT JOIN → всегда Hash Join
FULL JOIN → всегда Sort Merge Join
```

неверно.

Конкретную стратегию необходимо смотреть в физическом плане.

---

## Hints в Spark

Hints позволяют передать оптимизатору дополнительную информацию о желаемой стратегии.

В SQL:

```sql
SELECT /*+ BROADCAST(small_table) */
       *
FROM large_table l
JOIN small_table s
  ON l.id = s.id;
```

В DataFrame API:

```python
large_df.join(
    small_df.hint("broadcast"),
    "id"
)
```

### Основные hints

Наиболее распространённые:

* `BROADCAST`
* `MERGE`
* `SHUFFLE_HASH`
* `SHUFFLE_REPLICATE_NL`
* `REPARTITION`
* `COALESCE`
* `REBALANCE`

Конкретный набор и поведение hints зависят от версии Spark.

Важно понимать: hint — это **рекомендация для оптимизатора, а не абсолютная гарантия**, что физический план будет именно таким, как ожидается.

Поэтому после применения hint следует проверить:

```python
df.explain("formatted")
```

---

## Spark UI

Для анализа производительности Spark предоставляет **Spark UI**.

Основные разделы:

* Jobs
* Stages
* Tasks
* Executors
* SQL

Через UI можно увидеть:

* длительность stages;
* количество tasks;
* Shuffle Read;
* Shuffle Write;
* spill;
* загрузку executors;
* неравномерность выполнения tasks;
* физический план SQL-запроса.

При проблемах производительности Spark UI — один из главных инструментов диагностики.

---

## Spill

При выполнении shuffle, сортировок, агрегаций и JOIN Spark может использовать больше памяти, чем доступно executor.

В этом случае промежуточные данные могут быть временно записаны на диск.

Это называется **spill**.

Spill позволяет продолжить вычисление, но замедляет его из-за дискового I/O.

В Spark UI следует обращать внимание на:

* Memory Bytes Spilled
* Disk Bytes Spilled

Большой объём spill может указывать на:

* слишком большой объём данных;
* неудачное количество partitions;
* skew;
* недостаток памяти executor;
* неэффективную стратегию JOIN.

---

## Настройка ресурсов

В распределённом Spark-приложении важно не только написать корректный код, но и правильно выделить ресурсы.

Ключевые параметры:

```text
spark.executor.instances
spark.executor.cores
spark.executor.memory
spark.executor.memoryOverhead
```

При использовании dynamic allocation дополнительно:

```text
spark.dynamicAllocation.enabled
spark.dynamicAllocation.minExecutors
spark.dynamicAllocation.maxExecutors
spark.dynamicAllocation.initialExecutors
```

Например:

```text
4 executors
×
4 cores
=
16 параллельных tasks
```

В реальном кластере доступное количество CPU и памяти ограничивается ресурсами cluster manager.

---

## Dynamic Allocation

**Dynamic Allocation** позволяет автоматически изменять количество executors в зависимости от нагрузки.

Например:

```text
много pending tasks
        ↓
Spark добавляет executors

нагрузка снижается
        ↓
лишние executors удаляются
```

Включение:

```python
spark.conf.set(
    "spark.dynamicAllocation.enabled",
    "true"
)
```

Основные параметры:

```text
spark.dynamicAllocation.minExecutors
spark.dynamicAllocation.initialExecutors
spark.dynamicAllocation.maxExecutors
spark.dynamicAllocation.executorIdleTimeout
```

Dynamic Allocation особенно полезна в общем кластере, где ресурсы должны перераспределяться между приложениями.

При этом слишком агрессивное масштабирование может увеличить накладные расходы на запуск и удаление executors.

---

## Оптимизация количества partitions

Выбор количества partitions — один из важнейших аспектов настройки Spark.

### Слишком мало partitions

```text
мало tasks
↓
низкий параллелизм
↓
CPU простаивает
```

### Слишком много partitions

```text
много маленьких tasks
↓
scheduler overhead
↓
лишние операции
```

После shuffle количество partitions часто контролируется параметром:

```text
spark.sql.shuffle.partitions
```

Например:

```python
spark.conf.set(
    "spark.sql.shuffle.partitions",
    400
)
```

Подходящее значение зависит от:

* объёма данных;
* размера кластера;
* характера операции;
* количества CPU;
* skew.

Универсального числа, подходящего для всех задач, не существует.

---

## Cache и Persist

Если один и тот же DataFrame используется несколько раз, его можно сохранить:

```python
df_cached = df.cache()
```

или:

```python
df.persist()
```

Это позволяет избежать повторного вычисления предыдущих стадий.

Однако кэширование не является автоматическим ускорителем.

Если DataFrame используется один раз, `cache()` может только увеличить расход памяти и создать дополнительную работу.

Поэтому кэш следует использовать тогда, когда результат действительно будет переиспользован.

---

## Кэширование и уровни хранения

RDD поддерживает различные persistence levels, например:

```text
MEMORY_ONLY
MEMORY_AND_DISK
DISK_ONLY
```

Для DataFrame и Dataset также используются соответствующие механизмы persistence.

Выбор уровня зависит от доступной памяти и стоимости повторного вычисления.

---

## Пример SQL и PySpark

Одна из полезных практик обучения Spark — писать один и тот же запрос и на SQL, и на DataFrame API.

SQL:

```sql
SELECT
    d.department_name,
    AVG(s.salary) AS average_salary
FROM employees e
JOIN departments d
    ON e.department_id = d.department_id
JOIN salaries s
    ON e.employee_id = s.employee_id
WHERE s.salary >= 3000
GROUP BY d.department_name
ORDER BY average_salary DESC;
```

PySpark:

```python
result_df = (
    employees_df
    .join(departments_df, "department_id")
    .join(salaries_df, "employee_id")
    .filter(F.col("salary") >= 3000)
    .groupBy("department_name")
    .agg(
        F.avg("salary").alias("average_salary")
    )
    .orderBy(F.desc("average_salary"))
)
```

Оба варианта описывают одну и ту же логику, а Spark строит физический план уже самостоятельно.

---

## Практический подход к оптимизации Spark

При проблемах производительности полезно двигаться от данных к физическому плану.

### 1. Проверить объём данных

Сколько данных реально читает приложение?

Если запрос читает 2 TB только ради получения нескольких столбцов, сначала нужно разобраться с чтением данных, а не с executor memory.

### 2. Проверить план

```python
df.explain("formatted")
```

Ищем:

```text
Exchange
Sort
Join
Aggregate
Scan
```

Особенно внимательно смотрим на `Exchange`, поскольку он обычно означает shuffle.

### 3. Проверить Spark UI

Смотрим:

* самые долгие stages;
* Shuffle Read/Write;
* spill;
* распределение длительности tasks;
* размер обрабатываемых данных.

### 4. Проверить skew

Если несколько tasks существенно медленнее остальных, проверяем распределение ключей.

### 5. Уменьшить объём данных

До тяжёлых операций стараемся:

* фильтровать;
* выбирать только необходимые столбцы;
* агрегировать там, где это возможно.

### 6. Проверить JOIN

Для небольшой таблицы может подойти:

```python
broadcast(...)
```

Для больших таблиц необходимо анализировать shuffle и выбранную стратегию JOIN.

### 7. Проверить partitions

Следует избегать как слишком большого, так и слишком малого количества partitions.

---

## Основные анти-паттерны

### collect() для больших объёмов

```python
df.collect()
```

переносит результаты на Driver.

Если данных много, Driver может получить `OutOfMemoryError`.

Безопаснее использовать:

```python
df.show()
```

или ограничивать результат:

```python
df.limit(100).collect()
```

---

### Ненужный cache()

```python
df.cache()
```

не следует использовать автоматически.

Кэширование оправдано, когда DataFrame повторно используется и стоимость его повторного вычисления выше стоимости хранения.

---

### Ненужный repartition()

Каждый `repartition()` потенциально создаёт shuffle.

Поэтому не стоит добавлять:

```python
df.repartition(...)
```

без понимания, зачем именно он нужен.

---

### Большой Broadcast

Broadcast небольшой таблицы может сильно ускорить JOIN.

Broadcast большой таблицы может привести к:

```text
executor OOM
```

Поэтому размер таблицы всегда необходимо учитывать.

---

### Игнорирование skew

Даже увеличение числа executors не всегда решает проблему skew.

Если 90% данных попали в один ключ, одна task всё равно может остаться значительно тяжелее остальных.

---

## Что нужно помнить

Основную модель Spark можно свести к следующей цепочке:

```text
Spark Application
        ↓
      Driver
        ↓
Logical Plan
        ↓
Catalyst
        ↓
Physical Plan
        ↓
      Job
        ↓
     Stages
        ↓
      Tasks
        ↓
    Executors
        ↓
Partitions / Shuffle
```

Ключевые понятия:

* **Driver** — управляет приложением.
* **Executor** — выполняет задачи.
* **Partition** — часть данных.
* **Task** — обработка одной partition.
* **Job** — работа, запущенная action.
* **Stage** — часть Job между shuffle-границами.
* **Shuffle** — перераспределение данных между partitions.
* **Skew** — неравномерное распределение данных.
* **Catalyst** — оптимизатор Spark SQL.
* **AQE** — адаптивная оптимизация физического выполнения.

Самая полезная практическая привычка при работе со Spark — не пытаться оптимизировать приложение вслепую. Сначала посмотреть `explain()`, затем Spark UI, определить конкретное узкое место и только после этого менять JOIN, partitions, память или конфигурацию кластера.

---

## Где изучать Spark

Для практики полезно регулярно решать одну и ту же задачу несколькими способами:

```text
SQL
↓
PySpark DataFrame API
↓
план выполнения
↓
анализ Spark UI
```

Это позволяет одновременно развивать:

* SQL;
* PySpark;
* понимание оптимизатора;
* навыки анализа распределённых вычислений.

Полезный формат обучения — брать реальные SQL-запросы и переписывать их на PySpark, а затем проверять, совпадает ли результат и как отличается физический план.

Для практики задач на PySpark можно использовать:

* [StrataScratch](https://platform.stratascratch.com/coding?code_type=6)

Для самостоятельного изучения Spark также полезно развернуть локальный кластер или использовать Docker.

---

## Итог

Apache Spark — это не просто «быстрый Python для больших данных». Это распределённый движок, который:

1. разбивает данные на partitions;
2. строит логический и физический планы;
3. делит вычисления на jobs, stages и tasks;
4. распределяет tasks между executors;
5. выполняет операции параллельно;
6. использует shuffle, когда данные необходимо перераспределить;
7. оптимизирует выполнение с помощью Catalyst и AQE.

Поэтому для эффективной работы со Spark недостаточно знать только синтаксис `select()`, `filter()` и `groupBy()`. Необходимо понимать, **как данные перемещаются между узлами, где возникают shuffle и skew, как выбирается стратегия JOIN и как всё это выглядит в физическом плане и Spark UI**.
