# CLICKHOUSE

---

## Введение

Первое что нужно запомнить **ClickHouse** - это колоночно-ориентированная БД, т.е. данные считываются по необходимым нам колонкам. В свою очередь, это позволяет считывать и агрегировать большой объем данных за короткий промежуток времени.

---

## Колоночная ориентация

Физически, колоночная ориентация представляет каждую колонку в виде отдельного файла, из-за чего считывать данные через команду `SELECT *` - значит считывать каждый файл таблицы.

![table_col](../png/clickhouse_1.jpg)

Колоночная ориентация имеет ряд преимуществ:

* Оптимизация хранения данных.
* Быстрое чтение большого количества данных
* Быстрое выполнение агрегационных запросов (OLAP-нагрузка)

Но есть и ряд недостатков:
* Плохо подходит под OLTP нагрузку, т.е. изменять, удалять или считывать единичные строки может быть затратнее по времени
* Ограниченная поддержка ACID
* Сложность управления

---

## Как работает ClickHouse

Главной особенностью быстрой работы ClickHouse является использование **разреженного индекса**. Отличие данного индекса от обычного B-tree индекса состоит в том, что помечается не каждое значение колонки, а с каким-либо промежутком (по умолчанию оно равно 8192, но можно изменить с помощью параметра `index_granularity`). В ClickHouse данный промежуток значений называется **гранулой**. Если говорить более научно, то **гранула** - это логическое разделение строк внутри блока. Такой индекс по умолчанию создается при создании таблиц семейства MergeTree (по умолчанию, если мы не говорим об определенном движке, будет подразумеваться именно он). Разреженный индекс является отдельным объектом, значения которого являются первым значением каждой гранулы, и который ссылается на файл с засечками.

![index_gr](../png/clickhouse_2.jpg)

Важно помнить, что при поднятии кластера файл с индексами всегда считывается в оперативную память и храниться именно там (соответственно если вам не хватит места в оперативной памяти то кластер даже не поднимется).

---

## Типы данных, создание таблиц и полей, движки таблиц

### Типы данных

ClickHouse поддерживает достаточно большое количество типов данных. Мы рассмотрим только часть из них:

#### Числовые типы

* **Int8**, **Int16** ... **Int256** - целые числа фиксированной длины со знаком от 8 до 256 разрядов.

* **UInt8**, **UInt16** ... **UInt256** - целые числа фиксированной длины без знака от 8 до 256 разрядов.

В ClickHouse рекомендуется зачастую использовать целые числа, так как они занимают значительно меньше места, чем например строки, за счет чего происходит ускорение работы. Например, намного эффективнее хранить **1234** метра, чем **1,234** километра.

* **Float32**, **Float64** - числа с плавающей запятой. Эти типы прекрасно подходят для различных статистических математических расчётов. Из-за особенностей данных типов не рекомендуется использовать в точных вычислениях, например финанасах.

Пример особенности, если выполнить следующий запрос в клике:

```
SELECT 1 - 0.9::float 
```

то в итоге мы получим ошибку округления, а именно **0.10000002384185791**

* **Decimal32(S)** ... **Decimal256(S)** - дробные числа с сохранением точности операций, где S - это количество знаков после запятой. Данный тип уже подойдет для точных вычислений 100%

#### Строковые типы

* **String** - набор байт произвольной длины. 

* **FixedString** - строка фиксированной длины N байт.

#### Типы для работы с датой

* **Date** и **Date32** - предназначен для хранения дат. Различие заключается в поддерживаемом диапазоне дат.

* **DateTime** и **DateTime64** - позволяет хранить даты и время с заданной точностью до секунд, в случае DateTime64 точность до наносекунд. 

#### Дополнительные типы данных

*  **UUID** (универсальный уникальный идентификатор) - это 16-байтовое число, используемое для идентификации записей.

* **IPv4**, **IPv6** - типы, предназначенные для хранения IP-адресов.

На самом деле в документации таких типо данных намного больше, но эти я выбрал в качестве примера.

#### Композитные типы данных

Композитные типы данных хранят в себе более сложные структуры, такие как массивы, кортежи и т.д.

* **Array (Массив)** - хранит в себе N-ое кол-во элеметов любого типа данных. Необходимо помнить, что массивы не могут содержать разные типы данных. Первый элемент начинает свою индексацию с 1, в отличии от общепринятой индексации с 0.

* **Tuple (Кортеж)** - этот тип данных используется для хранения связанных между собой данных. Например информация о клиентах.

* **Вложенная структура** (Nested Structure) - по сути это таблица в таблице.

* **Map** - данный тип похож на словарь в питоне, хранит пары ключ-значение.

* **Enum** - позволяет определить ограниченное количество возможных значений, присваиваемых столбцу, и хранить данные компактнее, чем при использовании строк.

```sql
drop table if exists type_data;
CREATE TABLE type_data
(
--------------------------
-- основные типы данных --
--------------------------
    i Int8,             -- Int8-256 (со знаком)
    ui UInt8,           -- UInt8-256 (без знака)
    fl Float32,         -- Float32-64 (для мат.расчетов, но не для финансов)  
    dc Decimal32(5),    -- Decimal32-256 (точность после запятой)
    st String,          -- имеет произвольную длинну
    fst FixedString(5), -- имеет фиксированную длинну
--------------------------------
-- дополнитеьлные типы данных --
--------------------------------
    UID UUID, -- уникальный идентификатор
    ip4 IPv4, -- 127.0.0.1
    ip6 IPv6, -- f2c6:e19b:da60:52ad:2cef:62fe:0279
--------------------------------
--    типы даты и времени     --
--------------------------------
    dt Date,          -- Date32 (различаются диапозном дат)
    dtm DateTime,     -- Сохрняет время с точностью то секунд
    dtm64 DateTime64, -- Сохрняет время с точностью то наносекунд
------------------------------------
--    композитные типы данных     -- позволяют хранить более сложные структуры данных
------------------------------------
    ar Array(UInt8),                      -- массив данных
    tu Tuple(Date, UInt16, Decimal32(2)), -- кортеж
    ns Nested(                            -- вложенные структуры
            col1 String,
            col2 UInt64,
            col3 String
            ),
    mp Map(String, Int16), -- хранит в данные в виде ключ -> значение
    en Enum('bad' = 2,     -- хранит данные определенного значения
            'udovlet' = 3, 
            'good' = 4 )      
)
ENGINE = Log;
INSERT INTO type_data
(
    i, ui, fl, dc, st, fst,
    UID, ip4, ip6,
    dt, dtm, dtm64,
    ar, tu, 
    ns.col1, ns.col2, ns.col3,
    mp, en
)
VALUES 
(
    -100,                							-- i Int8
    200,                						    -- ui UInt8
    3.14,                							-- fl Float32
    123.45,             							-- dc Decimal32(5)
    'Пример строки',     							-- st String
    'ABCDE',             							-- fst FixedString(5)
    generateUUIDv4(),    							-- UID UUID
    '192.168.1.1',      						    -- ip4 IPv4
    '2001:db8::1',       							-- ip6 IPv6
    toDate('2025-04-30'),                      		-- dt Date
    toDateTime('2025-04-30 14:30:00'),         		-- dtm DateTime
    toDateTime64('2025-04-30 14:30:00.123456', 6), 	-- dtm64 DateTime64
    [10, 20, 30],        							-- ar Array(UInt8)
    (toDate('2025-04-30'), 150, 99.99), 			-- tu Tuple(Date, UInt16, Decimal32(2))
    ['one'],             							-- ns.col1 Array(String)
    [123456],            							-- ns.col2 Array(UInt64)
    ['value1'],          							-- ns.col3 Array(String)
    {'key1': 10, 'key2': -20}, 						-- mp Map(String, Int16)
    'udovlet'              							-- en Enum
);
select 
    ar[1],      -- обращение к элементам массива
    ar.size0,   -- получение размера массива
    tu,         -- чтение кортежа
    ns.col1,    -- обращение к вложенной структуре
    mp['key1'], -- получение данных из Map
    en          -- Чтение Enum
from type_data
```

### Приведение к типам данных

В ClickHouse можно приводить к типам данных разными способами, я покажу 3:

1. С помощью символов `"::"`. Очень знакомое приведение для тех, кто работал с PostgreSQL.
2. С помощью функции `CAST`. Ну в общем и целом это стандарт языка SQL
3. С промощью функций, разработанных для ClickHouse.

Первые 2 метода не рекомендуется использовать, как минимум потому, что они не обрабатывают пустые значения **NULL** в колонке. А это бывает критично. 

Возьмите за практику всегда использовать 3й вариант, как минимум потому что он имеет свои приятные фишечки.

Данные функции очень легко запомнить, так как имеют единый шаблон:

```
to<тип данных><разрядность, если такая имеется><модификатор, если это необходимо>
```

Модификатор позволяет вернуть какое-то значение, даже если не удалось выполнить преобразование к нужному типу.

Существует 3 модификатора:

* **OrNull** - если не удалось преобразовать в требуемый тип, возвращает NULL.
* **OrZero** - если не удалось преобразовать, возвращает 0.
* **OrDefault** - если не удалось преобразовать, возвращает значение по умолчанию.

```sql
DROP TABLE IF EXISTS cast_type_data;
CREATE TABLE cast_type_data (
    col String
)
ENGINE = Log;
INSERT INTO cast_type_data values ('1'),('2'),('1a'),('-1');
select CAST(NULL as Int64); -- не кастонётся
select
    col,
    toInt64OrNull(col),
    toInt8OrZero(col),
    toInt8OrDefault(col, -100),
    toUInt8(-1), toUInt8(-1.1), toUInt8(256) -- выход за пределы преобразует в значение по модулю диапазона
from cast_type_data;
```

В примере особое внимание уделите данной строке.

```sql
toUInt8(-1), toUInt8(-1.1), toUInt8(256)
```

Заметим, что выход за пределы диапозона **не выдаст ошибку**, а преобразует данные по модулю диапозона.

### Создание таблиц

#### Основное

Создание таблиц в Clickhouse делается абсолютно также, как и в других БД за исключением того, что в ClickHouse необходимо указывать **движок** таблицы, например:

```sql
CREATE TABLE <имя таблицы> (
    Id String
    ...
)
engine=MergeTree
PRIMARY KEY (Id, ...)
OREDR BY (Id, ...)
```

О движках таблиц можно узнать либо из [официльной документации](https://clickhouse.com/docs/ru/engines/table-engines) ClickHouse, также они будут подробнее рассмотрены далее. Здесь мы поверхностно рассмотрим синтаксис самого ходового движка MergeTree.

Основная идея MergeTree состоит в том, что данные записываются по частям, небольшими кусками, а потом объединяются в фоновом режиме. Физически это выглядит следующим образом:

![ch_merge](../png/clickhouse_3.gif)

Имя файла у таких таблиц состоит из следующих частей:

![ch_name](../png/clickhouse_4.png)

Обязательно при создании таблицы семейства MergeTree требует указать ключ сортировки через модификатор `ORDER BY`. По умолчанию он будет являться первичным ключом, однако можно указать другой первичный ключ через `PRIMARY KEY`. Также нужно запомнить, что если мы указываем `PRIMARY KEY`, то он должен начинаться абсолютно так же как и ORDER BY, иначе будет ошибка.

Чуть позже мы поговорим об еще одном параметре `PARTITION BY`, который позволяет разбить таблицу на партиции.

#### Атрибуты типов даннных при создании таблиц

Существует 2 ключевых атрибута: **LowСardinality** и **Nullable**.

* **LowCardinality** - меняет способ хранения низкокоординальных полей. Данный атрибут стоит использовать, когда в колонке содержится менее ста тысяч уникальных значений.

* **Nullable** - разрешает вставку со значением NULL. По умолчанию в поле с таким атрибутом вместо NULL будут вставляться значения по умолчанию, для строк это пустое значение, для чисел это 0.

```sql
drop table nl_lc_tabl;
CREATE TABLE nl_lc_tabl (
    a Nullable(UInt32),         -- разрешает вставку с пропуском значения.
    b LowCardinality(String),   -- ускоряет работу малокоординальных данных(часто повторяющихся)
    c UInt32
) ENGINE = MergeTree 
ORDER BY tuple(); -- определяет порядок строк по порядку вставки данных
INSERT INTO nl_lc_tabl VALUES (NULL,'test' ,1);
INSERT INTO nl_lc_tabl VALUES (1,'test2',NULL); -- null вставится как 0. Если тип данных строка вставляется как пустое значение
INSERT INTO nl_lc_tabl VALUES (1, NULL, 3); 
INSERT INTO nl_lc_tabl VALUES (1, 'test2', 4); 
SET input_format_null_as_default = 0; -- параметр, отвечающий за вставку пустых значений. Выполняется совместно с командой на вставку
INSERT INTO nl_lc_tabl VALUES (1, 'sdasda', NULL)
SELECT 
	a, 
	b, 
	c 
from nl_lc_tabl;
SELECT 
	toTypeName(a), 
	toTypeName(b), 
	toTypeName(c) 
from nl_lc_tabl;
```

#### Модификаторы колонок

Рассмотрим пример создания таблицы:

```sql
CREATE TABLE test_fields
(
      col_default UInt64 DEFAULT 42
    , col_materialized UInt64 MATERIALIZED col_default * 33
    , col_alias UInt64 ALIAS col_default + 1
    , col_codec String CODEC(ZSTD(10))
    , col_comment Date COMMENT 'Some comment'
    , col_ttl UInt64 DEFAULT 10  TTL col_comment + INTERVAL 5 DAY
)
...
```

Здесь мы наблюдаем, что после типов данных стоят дополнительные конструкции - модификаторы, разберем их более подробнее.

* `DEFAULT` - если не передается значение, то будет вставлено значение по умолчанию.
* `MATERIALIZED` - Данное поле будет материализовано и физически вычисляться при добавлении данных
* `ALIAS` - все тоже самое, только вычисляется по факту вызова в запросе
* `CODEC` - предназначено для сжатия файлов
* `COMMENT` - можно оставить комментарий
* `TTL` - time to live, время жизни значения. В случае истечения времени, значение либо удалится, либо можно преобразовать в дефолтное значение. TTL нельзя применить к полям, входящим в ключ сортировки! Можно использовать только в движках MergeTree.

```sql
drop table if exists test_fields_without_ttl;
CREATE TABLE test_fields_without_ttl
(
     col_default UInt64 DEFAULT 42
    ,col_materialized UInt64 MATERIALIZED col_default * 33 -- к данной колонке можно обратиться только по имени
    ,col_alias UInt64 ALIAS col_default + 1                -- к данной колонке можно обратиться только по имени
    ,col_codec String CODEC(ZSTD(10))
    ,col_comment Date COMMENT 'Some comment'
)
ENGINE = Log;
INSERT INTO test_fields_without_ttl (
    col_default,
    col_codec,
    col_comment,
)
SELECT
    number,
    'какой-то текст +' || toString(number),
    toDate(now()) + number
FROM numbers(60);
select * from test_fields_without_ttl;
drop table if exists test_fields_with_ttl;
CREATE TABLE test_fields_with_ttl
(
      col_default UInt64 DEFAULT 42
    , col_materialized UInt64 MATERIALIZED col_default * 33
    , col_alias UInt64 ALIAS col_default + 1
    , col_codec String CODEC(ZSTD(10))
    , col_comment Date COMMENT 'Some comment'
    , col_ttl UInt64 DEFAULT 10  TTL col_comment + INTERVAL 5 DAY
)
ENGINE = MergeTree()
ORDER BY (col_default);
INSERT INTO test_fields_with_ttl (
    col_default,
    col_codec,
    col_comment,
    col_ttl
)
    SELECT
        number,
        'какой-то текст +' || toString(number),
        toDate(now()) - number,
        rand(1) % 100000000
    FROM numbers(20);
select 
      col_default
    , col_materialized 
    , col_alias 
    , col_codec 
    , col_comment
    , col_ttl
from test_fields_with_ttl;
SHOW FULL COLUMNS FROM test_fields_with_ttl;  -- тут можно увидеть комментарий к столбцу
```

Теперь выполним скрипт, описанный выше (его первую часть) и посмотрим, что физически создалось в файловой системе clickhouse контейнера.

![ch_cols](../png/clickhouse_5.png)

Мы видим, что создалось 4 колонки, что в целом и логично, колонка с модификаторм ALIAS должна будет вычисляться налету.

Теперь выполним запрос:

```sql
SELECT * FROM test_fields_without_ttl;
```

И увидим следующее:

![ch_no_cols](../png/clickhouse_6.png)

Другими словами, ни материализованную (хоть она физически существует), ни алиас мы вызвать через `SELECT *` не можем, а чтобы их вызвать, необходимо в SELECT обращаться напрямую по их именам.

---

## Партицирование таблиц

Партиция - это набор данных, объединённых с единым критерием, например по месяцу. Это позволяет ускорить процесс считывания данных по выбранному критерию при фильтрации.

![ch_partitions](../png/clickhouse_7.jpg)

В ClickHouse создавать партиции можно только на таблицах семейста MergeTree. Партиция указывается при создании таблицы с помощью ключевой конструкции `PARTITION BY`. Например:

```sql
CREATE TABLE partition_table
(
    ...
)
ENGINE MergeTree
ORDER BY (dt)
PARTITION BY toYYYYMM(dt)
PRIMARY KEY (dt);
```

Существует несколько типов партицирования:

* **range** - по диапазонам значений

```sql
CREATE TABLE table_range
(
    id UInt32,
    name String,
    created_at Date
)
ENGINE = MergeTree
PARTITION BY
    CASE
        WHEN id < 10000 THEN 'range_1'
        WHEN id < 20000 THEN 'range_2'
        ELSE 'range_3'
    END
ORDER BY id;
```

* **interval** - по интервалу значений

```sql
CREATE TABLE table_interval
(
    id UInt32,
    amount Float32,
    sale_date Date
)
ENGINE = MergeTree
PARTITION BY toYYYYMM(sale_date)
ORDER BY id;
```

* **Хеш-партицирование** - на основе хеш-функции

```sql
CREATE TABLE table_hash
(
    user_id UInt64,
    event String
)
ENGINE = MergeTree
PARTITION BY cityHash64(user_id) % 10
ORDER BY user_id;
```

* **Cоставное партицирование** - может включать несколько полей

```sql
CREATE TABLE table_composiste
(
    order_id UInt64,
    customer_id UInt64,
    order_date Date
)
ENGINE = MergeTree
PARTITION BY (toYYYYMM(order_date), customer_id % 10)
ORDER BY order_id;
```

Важно понимать, что при разделении таблиц по партициям данные сливаются в рамках своих партиций. Из-за этого случаются небольшие коллизии в данных на определённых движках. (Об этом мы поговорим в разделе о движках)

---

## Индексы

### Индекс основного назначения

С данными видами индексов мы уже познакомились. Это первичный ключ или ключ сортировки, который создается с таблицей и хранится в файле `primary.idx`. (разреженный индекс)

Данный ключ создается с помощью ключевых конструкций `ORDER BY (...)` или `PRIMARY KEY (...)`(набор полей должен повторять ключ сортировки и не может быть больше него).

Также напомним, что индекс всегда хранится в оперативной памяти.

### Индексы пропуска данных

Существует несколько типов индексов пропуска данных:

* **minmax** - индекс хранит минимальное и максимаьное значение для указанного поля. Подходит для дат и чисел.
* **set** - индекс хранит уникальные значения столбца в рамках блока. Имеет параметр `max_rows`, который задает блок, если значение равно **0**, то берется весь кусок данных. 
* **bloom_filter** - реализован на основе фильтра Блума для поиска по точному совподению. 
* **ngrambf_v1** - реализован на основе фильтра Блума для полнотекствого поиска по n-граммам
* **tokenbf_v1** - реализован на основе фильтра Блума для полнотекствого поиска по фразам

Синтаксис создания

```sql
ALTER TABLE <имя таблицы>
ADD INDEX <имя индекса> <имя колонки> TYPE <тип индекса> GRANULARITY <число>
```

Разберём первые 2 индекса.

#### Индекс minmax 

Индекс **minmax** требует корреляции с первичным ключом, т.е. по мере возрастания первичного ключа должна быть корреляция с колонкой, на которую мы хотим повесить индекс.

Данный индекс работает достаточно просто, тк берутся определенные гранулы, заданные с помощью параметра `GRANULARITY`, и в данном куске ищется минимальное и максимальное значение.

![ch_minmax](../png/clickhouse_6.png)

#### Индекс set

Аналогично тому, как в Python, если список [1,2,2,3,3,3] преобразовать в множество set(), только уникальные значения (1,2,3), берутся гранулы и удаляются дубликаты:

![ch_set_idx](../png/clickhouse_9.jpg)

---

## Движки

### SummingMergeTree

SummingMergeTree - это таблица с группировкой одинаковых записей по ключу сортировки и применением суммы к перечисленным полям

```sql
DROP TABLE IF EXISTS summing_mt;
 
CREATE TABLE summing_mt
(
    id UInt32,
    val UInt32,
    dt datetime,
    example UInt32  -- столбец, не входящий в ключ сортировки и параметры движка
)
ENGINE = SummingMergeTree(val) -- сумма будет считаться по полю val, так как оно указано в качестве параметра движка 
ORDER BY (id)
PARTITION BY toYYYYMM(dt); -- записи по этому ключу будут группироваться
 
INSERT INTO summing_mt
SELECT 
    number % 2, 
    (number + 1) * 10, 
    now() + number * 60 * 60 * 24,
    (number + 1) * 100 
from numbers(30);
```

Важно отметить, что данные сливаются только в рамках одной **партиции**.

Посмотрим на текущие данные

```sql
SELECT * FROM summing_mt;
```

А теперь добавим ещё данных и снова выведем их предыдущим запросом

```sql
INSERT INTO summing_mt
SELECT 
    number % 2, 
    (number + 1) * 10, 
    now() + number * 60 * 60 * 24,
    (number + 1) * 100 
from numbers(30);
```

Видим, что создался новый блок данных, и теперь эти блоки необходимо слить друг с другом. В движке происходит автоматическое суммирование данных в режиме "ленивого" слияния, произвести явно его можно командой

```sql
OPTIMIZE TABLE summing_mt FINAL; -- ручное слияние
```

Посмотрим ещё раз на данные. Теперь они слились в единое целое, посчиталась общая сумма. Чтобы посмотреть итоговый результат слияния, можно использовать ключевое слово `FINAL`

```sql
SELECT * FROM summing_mt FINAL
```

### AggregatingMergeTree

AggregatingMergeTree - это таблица, которая группирует одинаковые записи по ключу сортировки и применяет агрегатные функции к полям.

#### Комбинаторы агрегатных функций

У агрегаторов есть так называемые [**Комбинаторы агрегатных функций**](https://clickhouse.com/docs/ru/sql-reference/aggregate-functions/combinators).

Данные комбинаторы дают достаточно большой спектр творчества при работе с агрегационными функциями, как например в PostgreSQL можно делать так 

```sql
...
count(Distinct col1) filter(where col1 > 1)
...
```

так вот, в ClickHouse это можно сделать вот так:

```sql
...
countDistinctIf(col1, col1 > 1) 
...
```

Как ты понимаешь **-Distinct** и **-If** являются комбинаторами, их в целом очень много, с полным списком можно ознакомиться по ссылке.

#### Агрегаторные типы данных

Существуют 2 дополнительных типа данных **SimpleAggregateFunction** и **AggregateFunction**.

- **SimpleAggregateFunction** - предназначен для хранения простых агрегатов, который хранит конечное состояние
- **AggregateFunction** - сложные агрегаты, который хранит состояние всех добавленных значений

Теперь рассмотрим сам движок и как он работает с данными типами данных.

##### SimpleAggregateFunction

Создаем таблицу с 2мя простыми агрегационными функциями **MAX** и **SUM**. (в общем и целом простыми можно назвать все агрегаты, которые могут в себе хранить конечное значение блока и без коллизий агрегироваться с конечными значениями других блоков)

```sql
DROP TABLE IF EXISTS simple;

CREATE TABLE simple (
    id UInt64, 
    val_sum SimpleAggregateFunction(sum, UInt64), -- предусмотрен для хранения простых агрегатов(хранит конечное состояние)
    val_max SimpleAggregateFunction(max, UInt32)
) 
ENGINE=AggregatingMergeTree 
ORDER BY id;

INSERT INTO simple SELECT  1, number, number from numbers(10);
INSERT INTO simple SELECT  2, sum(number), max(number) from numbers(5);
INSERT INTO simple SELECT  1, number, number from numbers(8);
```

Выведем результат запроса и увидим, что движок агрегирует данные в блоке данных при вставке. В итоге мы увидим 3 строки

```sql
SELECT * FROM simple
```

Но нам же нужно для ID=1 получить 1 строку. Можно дождаться следующего слияния, а можно выполнить следующий запрос, который произведёт **логическое** слияние и выдаст конечный результат:

```sql
SELECT * FROM simple FINAL;
```

На больших таблицах, где данные поступают большими блоками и достаточно регулярно, намного эффективнее получить тот же результат следущим запросом:

```sql
SELECT 
    id, 
    sum(val_sum),
    max(val_max) 
FROM simple
GROUP BY id;
```

##### AggregateFunction

Здесь используются специальные комбинаторы.

Комбинаторы агрегаторных типов данных:

* SimpleState - возвращает результат агрегирующей функции типа SimpleAggregateFunction.

* State - возвращает промежуточное состояние типа AggregateFunction, используется при вставке.

* Merge - берёт множество состояний, объединяет их и возвращает результат полной агрегации данных.

* MergeState - выполняет слияние промежуточных состояний агрегации, возвращает промежуточное состояние агрегации.

По сути, данные комбинаторы хранят состояния данных и выполнить просто команду ```SELECT *``` к данным таблицам не получится. Давайте создадим такую таблицу и попробуем достать выборку.

```sql
DROP TABLE IF EXISTS aggr_func_tbl;

CREATE TABLE aggr_func_tbl
(
    id UInt64,
    val_uniq AggregateFunction(uniq, UInt64),         -- Хранит в себе промежуточное состояние данных
    val_any AggregateFunction(anyIf, String, UInt8),
    val_quant AggregateFunction(quantiles(0.5, 0.9), UInt64)
) ENGINE=AggregatingMergeTree 
ORDER BY id;

INSERT INTO aggr_func_tbl
SELECT 
    1, 
    uniqState(toUInt64(rnd)),                 -- кол-во уникальных значений
    anyIfState(toString(rnd),rnd%2=0),
    quantilesState(0.5, 0.9)(toUInt64(rnd)) 
FROM
    (SELECT arrayJoin(arrayMap(i -> i * 10, range(10))) as rnd);
```

Теперь достанем выборку:

```sql
SELECT * FROM aggr_func_tbl
```

Произошла ошибка. Чтобы получить какой-то результат, можно выполнить следующую команду:

```sql
SELECT * FROM aggr_func_tbl  FORMAT Vertical
```

А теперь посмотрим на данные, но их невозможно прочитать. Чтобы достать необходимые для нас данные необходимо воспользоваться комбинатором **-Merge**, вот так:

```sql
SELECT uniqMerge(val_uniq), 
        quantilesMerge(0.5, 0.9)(val_quant), 
        anyIfMerge(val_any) 
FROM aggr_func_tbl
```

### ReplacingMergeTree

Данный движок убирает дубликаты строк с одинаковым ключом сотрировки в блоке и во время слияния блоков.

Создаем таблицу и наполняем ее данными.

```sql
DROP TABLE IF EXISTS replacing_merge_tree;


CREATE TABLE replacing_merge_tree
(
    id UInt32,
    dt date,
    val String
)
ENGINE = ReplacingMergeTree
ORDER BY (id); -- ключ по которому удаляются дубликаты

INSERT INTO replacing_merge_tree
VALUES (1, '2025-01-01', 'Djo'), (2, '2025-01-01', 'JB'),(3, '2025-01-01', 'JD');
```

Теперь посмотрим на данные:

```sql
SELECT * FROM replacing_merge_tree
```

Получили 3 строки. Давайте теперь еще добавим пару строк, а потом посмотрим на результат.

```sql
INSERT INTO replacing_merge_tree
VALUES (1, '2025-01-02', 'Djo'), (1, '2025-01-03', 'Djo');

SELECT * FROM replacing_merge_tree;
```

И что мы видим. Добавилась 1 из данных записей, потому что дубликаты по ключу удаляются в одном блоке и при слиянии. Давайте запустим ручное слияние и у нас останется только 3 строки.

```sql
OPTIMIZE TABLE replacing_merge_tree;
```

Но что, если нам нужно всегда оставлять только последнее значение. Для этого в движок можно передать колонку, относительно которой Clickhouse будет оставлять последнюю версию строки. Еще раз создадим новую таблицу:

```sql
DROP TABLE IF EXISTS replacing_merge_tree_with_version;

CREATE TABLE replacing_merge_tree_with_version
(
    id UInt32,
    dt date,
    val String
)
ENGINE = ReplacingMergeTree(dt) -- колонка по которой необходимо оставить последнее
                                -- значение. Может быть либо числовой, либо датой.
ORDER BY (id); -- ключ по которому удаляются дубликаты

INSERT INTO replacing_merge_tree_with_version -- Добавляем первый блок данных
VALUES (1, '2025-01-01', 'Djo'), (2, '2025-01-01', 'JB'),(3, '2025-01-01', 'JD');

INSERT INTO replacing_merge_tree_with_version -- Добавлем 2й блок данных
VALUES (1, '2025-01-02', 'Djo'), (1, '2025-01-03', 'Djo');

OPTIMIZE TABLE replacing_merge_tree_with_version FINAL; -- Выполняем ручное слияние

SELECT * FROM replacing_merge_tree_with_version; -- смотрим и видим, что у ID=1, дата стоит 3 января 2025
```

Таким образом можно поддерживать акутальное состояние данных.

### CollapsingMergeTree

**CollapsingMergeTree** - удаляет дубликаты строк по ключу сортировки в зависимости от флага. 

Наглядным примером будет приложение с книгами, где необходимо запоминать на какой странице остановился пользователь. Создадим такую таблицу.

```sql
DROP TABLE IF EXISTS Books;

CREATE TABLE Books
(
    ID UInt64,
    Page UInt8,
    Sign Int8 -- имеет только 2 значения
)
ENGINE = CollapsingMergeTree(Sign) -- задаем флаг
ORDER BY ID
```

Пользователь открывает книгу 1 на странице 1:

```sql
INSERT INTO Books values (1, 1, 1);
```

Далее перелистывает на страницу 2:

```
INSERT INTO Books values (1, 1, -1),(1, 2, 1);
```

Выполним логическое слияние блоков и сразу выведем результат на экран:

```sql
SELECT * FROM Books FINAL
```

Остаётся актуальная страница книги

### Log

Ничем не примечательный движок, предназначен для тестов и хранения небольших объёмов данных. Каждая колонка такого движка хранится в отдельном файле.

```sql
DROP TABLE IF EXISTS el;

CREATE TABLE el
(
    id UInt32,
    dt date
)
ENGINE = Log;

INSERT INTO el
select 
number,
now()::date + number,
from numbers(10);

SELECT * FROM el;
```

### File

File - позволяет записывать данные в формате файла

```sql
DROP TABLE IF EXISTS ef;

CREATE TABLE ef
(
    id UInt32,
    dt date
)
ENGINE = File(CSV);

INSERT INTO ef
SELECT 
    number,
    now()::date + number
FROM numbers(10);

SELECT * FROM ef;
```

Данная таблица сохранится как CSV файл, где находятся все таблицы внутри самого Clickhouse (путь к каталогу **/var/lib/clickhouse/data/ef/**).

### Buffer

Если у вас большое количество небольших вставок данных, то для ускорения процесса, во избежание частых вызовов слияния, необходимо использовать движок **Buffer**. Он сохраняет вставки в оперативную память, после чего данные сливаются на диск в другую таблицу.

```sql
DROP TABLE IF EXISTS eb;

DROP TABLE IF EXISTS ebt;

CREATE TABLE ebt
    (
    id UInt32,
    dt date
)
ENGINE = Log;

CREATE TABLE eb
(
    id UInt32,
    dt date
)
ENGINE = Buffer(db,      -- имя БД
                ebt,     -- имя таблицы для слива данных
                16,      -- параллелизм (рекомендация 16)
                30,      -- минимальное время слияния
                60,      -- минимальное время слияния
                5,       -- минимальное кол-во строк для слияния
                10,      -- максимальное кол-во строк для слияния
                10000,   -- минимальное кол-во байт для слияния
                10000    -- максимальное кол-во байт для слияния
                );

INSERT INTO eb
SELECT 
    number,
    now()::date + number
FROM numbers(1);
```

Читать данные из буферной таблицы не получится.

```sql
SELECT * FROM eb; -- выдаст ошибку
```

Выборку можно получить только из таблицы куда сливаются данные.

```sql
SELECT * FROM ebt;
```

Чтобы слить данные из ОП в таблицу вручную, используйте следующий запрос:

```sql
OPTIMIZE TABLE eb;
```

### Memory

В движке Memory данные хранятся только в оперативной памяти, поэтому при перезапуске CH данные будут утеряны. Данный движок часто используется при оптимизации запроса.

```sql
DROP TABLE IF EXISTS em;

CREATE TABLE em
(
    id UInt32,
    dt date
)
ENGINE = Memory;

INSERT INTO em
SELECT 
    number,
    now()::date + number
FROM numbers(100);

SELECT * FROM em;
```

### Set

Движок `set` предназначен для использования в правой части оператора IN. Не хранит дублирующие значения. Предназначен для оптимизации запросов.

```sql
DROP TABLE IF EXISTS es;

CREATE TABLE es
    (
        id UInt32
    )
    ENGINE = Set
SETTINGS persistent = 1; -- данные будут считываться из ОП.

INSERT INTO es SELECT number from numbers(30);

-- Создание 2й таблицы
DROP TABLE IF EXISTS est;

CREATE TABLE est
(
    id UInt32
)
ENGINE = MergeTree
ORDER BY (id);

INSERT INTO est SELECT number from numbers(300);
```

Читать данные из  Set-таблицы не получится. 

```sql
SELECT * FROM es -- выдаст ошибку 
```

Применять можно только таким образом:

```sql
SELECT *
FROM est
WHERE id in es
```

Также, такая таблица может состоять из нескольких колонок, чтобы правильно к ней обратиться, необходимо использовать следующий шаблон:

```sql
SELECT *
FROM table_name
WHERE (col1, col2, ..., colN) in table_set
```

### GenerateRandom

**GenerateRandom** - предназначен для генерации данных в СН с целью дальнейших тестов.

```sql
DROP TABLE IF EXISTS eg;

CREATE TABLE eg
(
    id UInt32, 
    val String,
    dt date,
    a Float32,
    b UUID,
    c Bool,
    d IPv6,
    e IPv4,
    g Array(UInt32)
)
ENGINE = GenerateRandom;
```
```sql
select * 
from eg
limit 10;
```

Можно использовать любой тип данных.

### PostgreSQL

**PostgreSQL** - позволяет из CH подключаться к PSQL и применять функции CH над данными из PSQL.

```sql
DROP TABLE IF EXISTS postgresql_table;

CREATE TABLE postgresql_table
(
    dept_id Int32,
    dept_name String,
    location String
)
ENGINE = PostgreSQL(
                    'postgres_server:5432', -- сервер, порт
                    'postgres',             -- БД 
                    'table_name',           -- таблица
                    'postgres',             -- логин
                    'postgres',             -- пароль
                    'public'                -- имя схемы
                    );

select * from postgresql_table;
```

Если запросы простые и PSQL может с ними справиться самостоятельно, то в CH транслируется только результат, если используются специфичные для CH функции, то PSQL транслирует все данные.

### Kafka

С помощью движка **Kafka** можно как отправлять данные в кафку, так и получать их из неё.

#### Отправка данных

Сейчас мы будем транслировать данные в кафку. Для начала создадим таблицу с нужным движком:

```sql
DROP TABLE IF EXISTS kafka_out_message;

 CREATE TABLE kafka_out_message
(
    id UInt32
)
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:29093',              -- подключение к брокеру
    kafka_topic_list = 'clickhouse.topic',          -- создай такой же топик
    kafka_group_name = 'clickhouse_consumer_group',
    kafka_format = 'JSONEachRow';
```

Чтобы транслировать данные в кафку необходимо выполнить обычную команду на вставку:

```sql
INSERT INTO kafka_out_message
SELECT number FROM numbers(30);
```

#### Получение данных

Это несколько сложнее. Создадим очередную таблицу:

```sql
DROP TABLE IF EXISTS kafka_input_stage;

CREATE TABLE kafka_input_stage
    (
        json_kafka String       -- задается только 1 строка с полным JSON-ом из кафки
    )
ENGINE = Kafka
SETTINGS
    kafka_broker_list = 'kafka:29093',
    kafka_topic_list = 'source.public.order_events',
    kafka_group_name = 'clickhouse_consumer_group',
    kafka_format = 'JSONAsString';  -- достаточно часто используется именно этот формат
```

Считать данные из этой таблицы просто так не получится, данная таблица сохраняет данные в памяти и может только перенаправлять данные в нужную нам таблицу с определённым методом хранения (движком).

Если все-таки хочется проверить, а получаем ли мы что-то с данного топика, необходимо выполнить следующий запрос:

```sql
SET stream_like_engine_allow_direct_select = 1
SELECT * FROM kafka_input_stage;
```

Теперь создаем таблицу, в которой будут непосредственно храниться данные:

```sql
DROP TABLE IF EXISTS table_stage_from_kafka;

CREATE TABLE table_stage_from_kafka 
(
    json_kafka String
)
Engine = Log
```

Так же нужно создать **материализованное представление**, которое будет перенаправлять данные из таблицы с движном **Kafka** в таблицу с движком **Log**.

```sql
DROP VIEW IF EXISTS kafka_input_stage_mv;

CREATE MATERIALIZED VIEW kafka_input_stage_mv TO table_stage_from_kafka AS 
    SELECT
        json_kafka
    FROM kafka_input_stage;
```

Осталось только достать данные из JSON:

```sql
SELECT  
    JSONExtractInt(JSONExtractString(json_kafka, 'before'), 'id') as before_id,  -- JSONExtract<тип данных>(<JSON строка>, <ключ>)
    JSONExtractInt(JSONExtractString(json_kafka, 'before'), 'order_id') as before_order_id,
    JSONExtractString(JSONExtractString(json_kafka, 'before'), 'status') as before_status,
    JSONExtractUInt(JSONExtractString(json_kafka, 'before'), 'ts') as before_ts,
    JSONExtractInt(JSONExtractString(json_kafka, 'after'), 'id') as after_id,
    JSONExtractInt(JSONExtractString(json_kafka, 'after'), 'order_id') as after_order_id,
    JSONExtractString(JSONExtractString(json_kafka, 'after'), 'status') as after_status,
    JSONExtractUInt(JSONExtractString(json_kafka, 'after'), 'ts') as after_ts,
    JSONExtractString(json_kafka, 'op') as op,
    toDateTime(JSONExtractInt(json_kafka, 'ts_ms') / 1000) as dt  -- Перевод времени из timestamp в читаемый вид
FROM table_stage_from_kafka
```

### ReplicatedMergeTree

Данный движок позволяет положить копию таблицы на репликационный сервер. Данный сервер предназначен для повышения отказоустойчивости всего кластера. Важно помнить, что репликация не происходит моментально, согласованность основного и репликационного сервера происходит со временем и может произойти ситуация, когда на основном сервере произошел сбой, далее перешли на реплику, но данные на ней сохранились не в полном объеме. Это абсолютно нормально для ClickHouse.

Создается такая таблица с определёнными особенностями, при создании объекта необходимо использовать конструкцию **ON CLUSTER 'имя кластера'**. Это конструкция копирует объект на каждый сервер всего кластера.

```sql
CREATE TABLE events ON CLUSTER 'company_cluster' (   -- имя кластера можно задавать по имени и с помощью макроса '{cluster}'
        time DateTime,
        uid  Int64,
        type LowCardinality(String)
    )
    ENGINE = ReplicatedMergeTree('/clickhouse/tables/{cluster}/{shard}/events', '{replica}') -- путь может быть любой, необходимо смотреть как положено это делать в вашей команде.
    PARTITION BY toDate(time)
    ORDER BY (uid);
```

### Distributed

В продолжении рапределённых таблиц мы не можем не остановиться на теме шардирования кластера. Шардирование - это по свой сути обыкновенное разделение даных по серверам по определенному признаку для повышения вычислительной способности всего кластера, т.е. мы ускоряем работу выполенения запросов (но не всех! JOIN и оконные функции будут **ебать мозги** если распределить данные не по ключу соединения и не по ключую разбития окна(PARTITION BY), соотвествено).

Сама **Distributed** таблица не хранит в себе ничего, она исключительно ссылается на таблицу, которая хранит в себе данные. Для примера, как хранимую таблицу, мы возьмем из предыдущего запроса "events". Данная таблица будет хранить только те данные, которые ей даст **Distributed** таблица на определённой шарде.

Создадим Distributed таблицу:

```sql
CREATE TABLE events_distr ON CLUSTER 'company_cluster' AS events
ENGINE = Distributed('company_cluster', -- имя кластера, можно исполльзовать макрос
                     db,                -- имя БД
                     events,            -- имя таблицы
                     uid);              -- имя колонки по которой будет распределять данные
```

Конструкция **AS events**, говорит создай таблицу **events_distr**, с такими же полями как и таблица events. Очень важно, чтобы колонки совпадали.

```sql
INSERT INTO events_distr VALUES
    ('2020-01-01 10:00:00', 100, 'view'),
    ('2020-01-01 10:05:00', 101, 'view'),
    ('2020-01-01 11:00:00', 100, 'contact'),
    ('2020-01-01 12:10:00', 101, 'view'),
    ('2020-01-02 08:10:00', 100, 'view'),
    ('2020-01-03 13:00:00', 103, 'view');
```

Теперь считаем данные из распределённой таблицы (events_distr) и из локальной таблицы (events). Видим, что в локальной не хватает данных из-за того, что другие данные хранятся на другой шарде.
