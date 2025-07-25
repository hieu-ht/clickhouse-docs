---
title: 'BigQuery против ClickHouse Cloud'
slug: /migrations/bigquery/biquery-vs-clickhouse-cloud
description: 'Как BigQuery отличается от ClickHouse Cloud'
keywords: ['migrate', 'migration', 'migrating', 'data', 'etl', 'elt', 'BigQuery']
---

import bigquery_1 from '@site/static/images/migrations/bigquery-1.png';
import Image from '@theme/IdealImage';


# BigQuery против ClickHouse Cloud: Эквивалентные и различные концепции

## Организация ресурсов {#resource-organization}

Способ организации ресурсов в ClickHouse Cloud похож на [иерархию ресурсов BigQuery](https://cloud.google.com/bigquery/docs/resource-hierarchy). Мы описываем конкретные различия ниже, основываясь на следующей диаграмме, показывающей иерархию ресурсов ClickHouse Cloud:

<Image img={bigquery_1} size="md" alt="Организации ресурсов"/>

### Организации {#organizations}

Подобно BigQuery, организации являются корневыми узлами в иерархии ресурсов ClickHouse Cloud. Первый пользователь, которого вы настраиваете в своем аккаунте ClickHouse Cloud, автоматически назначается в организацию, принадлежащую пользователю. Пользователь может приглашать дополнительных пользователей в организацию.

### Проекты BigQuery против Сервисов ClickHouse Cloud {#bigquery-projects-vs-clickhouse-cloud-services}

В рамках организаций вы можете создавать сервисы, аналогичные проектам BigQuery, потому что хранимые данные в ClickHouse Cloud ассоциируются с сервисом. В ClickHouse Cloud доступно [несколько типов сервисов](/cloud/manage/cloud-tiers). Каждый сервис ClickHouse Cloud развертывается в определенном регионе и включает:

1. Группу вычислительных узлов (в настоящее время 2 узла для сервиса уровня Разработка и 3 для сервиса уровня Продакшн). Для этих узлов ClickHouse Cloud [поддерживает вертикальное и горизонтальное масштабирование](/manage/scaling#how-scaling-works-in-clickhouse-cloud), как вручную, так и автоматически.
2. Папку объектного хранилища, где сервис хранит все данные.
3. Конечную точку (или несколько конечных точек, созданных через интерфейс ClickHouse Cloud UI) — URL сервиса, который вы используете для подключения к сервису (например, `https://dv2fzne24g.us-east-1.aws.clickhouse.cloud:8443`)

### Наборы данных BigQuery против Баз данных ClickHouse Cloud {#bigquery-datasets-vs-clickhouse-cloud-databases}

ClickHouse логически группирует таблицы в базы данных. Подобно наборам данных BigQuery, базы данных ClickHouse являются логическими контейнерами, которые организуют и контролируют доступ к данным таблиц.

### Папки BigQuery {#bigquery-folders}

В ClickHouse Cloud в настоящее время нет концепции, эквивалентной папкам BigQuery.

### Резервирование слотов и квоты BigQuery {#bigquery-slot-reservations-and-quotas}

Как и резервирование слотов BigQuery, вы можете [настроить вертикальное и горизонтальное автоматическое масштабирование](/manage/scaling#configuring-vertical-auto-scaling) в ClickHouse Cloud. Для вертикального автоматического масштабирования вы можете установить минимальный и максимальный размер для памяти и ядер CPU вычислительных узлов для сервиса. Сервис будет масштабироваться по мере необходимости в этих пределах. Эти настройки также доступны на этапе первоначального создания сервиса. Каждый вычислительный узел в сервисе имеет одинаковый размер. Вы можете изменять количество вычислительных узлов в пределах сервиса с помощью [горизонтального масштабирования](/manage/scaling#manual-horizontal-scaling).

Кроме того, аналогично квотам BigQuery, ClickHouse Cloud предлагает управление параллелизмом, ограничение использования памяти и планирование I/O, позволяя пользователям изолировать запросы в классы нагрузки. Устанавливая ограничения на общие ресурсы (ядра CPU, DRAM, ввод-вывод диска и сети) для конкретных классов нагрузки, он обеспечивает, чтобы эти запросы не влияли на другие критические бизнес-запросы. Управление параллелизмом предотвращает избыточное использование потоков в сценариях с высоким количеством параллельных запросов.

ClickHouse отслеживает размеры байтов памяти на уровне сервера, пользователя и запроса, что позволяет гибко настраивать лимиты использования памяти. Переизбыток памяти позволяет запросам использовать дополнительную свободную память сверх гарантированной памяти, при этом обеспечивая лимиты памяти для других запросов. Кроме того, использование памяти для агрегации, сортировки и соединений может быть ограничено, позволяя использовать внешние алгоритмы, когда лимит по памяти превышен.

Наконец, планирование I/O позволяет пользователям ограничивать доступ к локальным и удаленным дискам для классов нагрузки на основе максимальной пропускной способности, активных запросов и политики.

### Права доступа {#permissions}

ClickHouse Cloud [управляет доступом пользователей](/cloud/security/cloud-access-management) в двух местах, через [облачную консоль](/cloud/get-started/sql-console) и через базу данных. Доступ к консоли осуществляется через интерфейс пользователя [clickhouse.cloud](https://console.clickhouse.cloud). Доступ к базе данных управляется через учетные записи пользователей базы данных и роли. Кроме того, пользователям консоли могут быть предоставлены роли в базе данных, которые позволяют пользователю консоли взаимодействовать с базой данных через нашу [SQL консоль](/integrations/sql-clients/sql-console).

## Типы данных {#data-types}

ClickHouse предлагает более детализированную точность в отношении чисел. Например, BigQuery предлагает числовые типы [`INT64`, `NUMERIC`, `BIGNUMERIC` и `FLOAT64`](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#numeric_types). В отличие от этого, ClickHouse предлагает несколько типов точности для десятичных, плавающих и целых чисел. С этими типами данных пользователи ClickHouse могут оптимизировать использование памяти и затраты на хранение, что приводит к более быстрым запросам и меньшему потреблению ресурсов. Ниже мы сопоставляем эквивалентный тип ClickHouse для каждого типа BigQuery:

| BigQuery | ClickHouse |
|----------|------------|
| [ARRAY](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#array_type)    | [Array(t)](/sql-reference/data-types/array)   |
| [NUMERIC](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#decimal_types)  | [Decimal(P, S), Decimal32(S), Decimal64(S), Decimal128(S)](/sql-reference/data-types/decimal)    |
| [BIG NUMERIC](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#decimal_types) | [Decimal256(S)](/sql-reference/data-types/decimal) |
| [BOOL](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#boolean_type)     | [Bool](/sql-reference/data-types/boolean)       |
| [BYTES](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#bytes_type)    | [FixedString](/sql-reference/data-types/fixedstring) |
| [DATE](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#date_type)     | [Date32](/sql-reference/data-types/date32) (с более узким диапазоном) |
| [DATETIME](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#datetime_type) | [DateTime](/sql-reference/data-types/datetime), [DateTime64](/sql-reference/data-types/datetime64) (узкий диапазон, высокая точность) |
| [FLOAT64](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#floating_point_types)  | [Float64](/sql-reference/data-types/float)    |
| [GEOGRAPHY](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#geography_type) | [Geo Data Types](/sql-reference/data-types/float) |
| [INT64](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#integer_types)    | [UInt8, UInt16, UInt32, UInt64, UInt128, UInt256, Int8, Int16, Int32, Int64, Int128, Int256](/sql-reference/data-types/int-uint) |
| [INTERVAL](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#integer_types) | NA - [поддерживается как выражение](/sql-reference/data-types/special-data-types/interval#usage-remarks) или [через функции](/sql-reference/functions/date-time-functions#addyears) |
| [JSON](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#json_type)     | [JSON](/integrations/data-formats/json/inference)       |
| [STRING](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#string_type)   | [String (bytes)](/sql-reference/data-types/string) |
| [STRUCT](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#constructing_a_struct)   | [Tuple](/sql-reference/data-types/tuple), [Nested](/sql-reference/data-types/nested-data-structures/nested) |
| [TIME](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#time_type)     | [DateTime64](/sql-reference/data-types/datetime64) |
| [TIMESTAMP](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-types#timestamp_type) | [DateTime64](/sql-reference/data-types/datetime64) |

При наличии нескольких вариантов типов ClickHouse учитывайте фактический диапазон данных и выбирайте минимально необходимый. Также рассмотрите возможность использования [подходящих кодеков](https://clickhouse.com/blog/optimize-clickhouse-codecs-compression-schema) для дальнейшего сжатия.

## Техники ускорения запросов {#query-acceleration-techniques}

### Первичные и внешние ключи и первичный индекс {#primary-and-foreign-keys-and-primary-index}

В BigQuery таблица может иметь [ограничения первичных и внешних ключей](https://cloud.google.com/bigquery/docs/information-schema-table-constraints). Обычно первичные и внешние ключи используются в реляционных базах данных для обеспечения целостности данных. Значение первичного ключа обычно уникально для каждой строки и не равно `NULL`. Каждое значение внешнего ключа в строке должно присутствовать в столбце первичного ключа таблицы первичного ключа или быть `NULL`. В BigQuery эти ограничения не обеспечиваются, но оптимизатор запросов может использовать эту информацию для лучшей оптимизации запросов.

В ClickHouse таблица также может иметь первичный ключ. Как и в BigQuery, ClickHouse не обеспечивает уникальность значений в столбце первичного ключа таблицы. В отличие от BigQuery, данные таблицы хранятся на диске [отсортированными](/guides/best-practices/sparse-primary-indexes#optimal-compression-ratio-of-data-files) по значениям столбца(ов) первичного ключа. Оптимизатор запросов использует этот порядок сортировки, чтобы избежать повторной сортировки, минимизировать использование памяти для соединений и включить короткое замыкание для ограничений limit. В отличие от BigQuery, ClickHouse автоматически создает [индекс (разреженный) первичный индекс](/guides/best-practices/sparse-primary-indexes#an-index-design-for-massive-data-scales) на основе значений столбца первичного ключа. Этот индекс используется для ускорения всех запросов, содержащих фильтры по столбцам первичного ключа. В настоящее время ClickHouse не поддерживает ограничения внешних ключей.

## Вторичные индексы (Доступны только в ClickHouse) {#secondary-indexes-only-available-in-clickhouse}

В дополнение к первичному индексу, созданному на основе значений столбцов первичного ключа таблицы, ClickHouse позволяет создавать вторичные индексы по столбцам, отличным от тех, что находятся в первичном ключе. ClickHouse предлагает несколько типов вторичных индексов, каждый из которых подходит для разных типов запросов:

- **Индекс фильтра Блума**:
  - Используется для ускорения запросов с условиями равенства (например, =, IN).
  - Использует вероятностные структуры данных для определения существования значения в блоке данных.
- **Индекс токенов фильтра Блума**:
  - Похож на индекс фильтра Блума, но используется для токенизированных строк и подходит для запросов полного текста.
- **Мин-Макс индекс**:
  - Хранит минимальные и максимальные значения столбца для каждой части данных.
  - Помогает пропускать чтение частей данных, которые не попадают в указанный диапазон.

## Индексы поиска {#search-indexes}

Аналогично [индексам поиска](https://cloud.google.com/bigquery/docs/search-index) в BigQuery, [индексы полнотекстового поиска](/engines/table-engines/mergetree-family/invertedindexes) могут быть созданы для таблиц ClickHouse на столбцах со строковыми значениями.

## Векторные индексы {#vector-indexes}

BigQuery недавно представил [векторные индексы](https://cloud.google.com/bigquery/docs/vector-index) как функцию Pre-GA. Аналогично, ClickHouse имеет экспериментальную поддержку [индексов для ускорения](/engines/table-engines/mergetree-family/annindexes) случаев использования векторного поиска.

## Партиционирование {#partitioning}

Подобно BigQuery, ClickHouse использует партиционирование таблиц для повышения производительности и управляемости больших таблиц, разделяя таблицы на более мелкие, более управляемые части, называемые партициями. Мы подробно описываем партиционирование ClickHouse [здесь](/engines/table-engines/mergetree-family/custom-partitioning-key).

## Кластеризация {#clustering}

С помощью кластеризации BigQuery автоматически сортирует данные таблицы на основе значений нескольких указанных столбцов и размещает их в оптимально размеренных блоках. Кластеризация улучшает производительность запросов, позволяя BigQuery лучше оценивать стоимость выполнения запроса. С кластеризованными столбцами запросы также исключают сканирование ненужных данных.

В ClickHouse данные автоматически [кластеризуются на диске](/guides/best-practices/sparse-primary-indexes#optimal-compression-ratio-of-data-files) на основе столбцов первичного ключа таблицы и логически организуются в блоки, которые могут быть быстро найдены или отсеяны запросами, использующими структуру данных первичного индекса.

## Материализованные представления {#materialized-views}

Как BigQuery, так и ClickHouse поддерживают материализованные представления — заранее вычисленные результаты на основе результата трансформационного запроса против базовой таблицы для повышения производительности и эффективности.

## Запросы к материализованным представлениям {#querying-materialized-views}

Материализованные представления BigQuery можно запрашивать напрямую или использовать оптимизатором для обработки запросов к базовым таблицам. Если изменения в базовых таблицах могут аннулировать материализованное представление, данные считываются напрямую из базовых таблиц. Если изменения в базовых таблицах не аннулируют материализованное представление, то остальная информация считывается из материализованного представления, и только изменения считываются из базовых таблиц.

В ClickHouse материализованные представления можно запрашивать только напрямую. Однако по сравнению с BigQuery (в котором материализованные представления автоматически обновляются в течение 5 минут после изменения базовых таблиц, но не чаще чем [раз в 30 минут](https://cloud.google.com/bigquery/docs/materialized-views-manage#refresh)), материализованные представления всегда синхронизированы с базовой таблицей.

**Обновление материализованных представлений**

BigQuery периодически полностью обновляет материализованные представления, выполняя трансформационный запрос представления против базовой таблицы. Между обновлениями BigQuery объединяет данные материализованного представления с новыми данными базовых таблиц для обеспечения согласованных результатов запросов, при этом продолжая использовать материализованное представление.

В ClickHouse материализованные представления обновляются инкрементально. Этот механизм инкрементного обновления обеспечивает высокую масштабируемость и низкие вычислительные затраты: инкрементно обновляемые материализованные представления специально разработаны для сценариев, когда базовые таблицы содержат миллиарды или триллионы строк. Вместо повторных запросов к постоянно растущей базовой таблице для обновления материализованного представления ClickHouse просто вычисляет частичный результат из (только) значений вновь добавленных строк базовой таблицы. Этот частичный результат инкрементально объединяется с ранее рассчитанным частичным результатом в фоновом режиме. Это приводит к значительно более низким вычислительным затратам по сравнению с повторным обновлением материализованного представления из всей базовой таблицы.

## Транзакции {#transactions}

В отличие от ClickHouse, BigQuery поддерживает транзакции с несколькими предложениями внутри единого запроса или между несколькими запросами при использовании сеансов. Транзакция с несколькими предложениями позволяет вам выполнять изменяющие операции, такие как вставка или удаление строк в одной или нескольких таблицах, и либо зафиксировать, либо отменить изменения атомарно. Транзакции с несколькими предложениями находятся на [дорожной карте ClickHouse на 2024 год](https://github.com/ClickHouse/ClickHouse/issues/58392).

## Агрегатные функции {#aggregate-functions}

По сравнению с BigQuery, ClickHouse поставляется с значительно большим количеством встроенных агрегатных функций:

- BigQuery предлагает [18 агрегатных функций](https://cloud.google.com/bigquery/docs/reference/standard-sql/aggregate_functions) и [4 аппроксимирующие агрегатные функции](https://cloud.google.com/bigquery/docs/reference/standard-sql/approximate_aggregate_functions).
- ClickHouse имеет более [150 предустановленных функций агрегации](/sql-reference/aggregate-functions/reference), а также мощные [комбинаторы агрегации](/sql-reference/aggregate-functions/combinators) для [расширения](https://www.youtube.com/watch?v=7ApwD0cfAFI) поведения предустановленных агрегатных функций. Например, вы можете применять более 150 предустановленных агрегатных функций к массивам, а не к строкам таблицы, просто вызвав их с помощью [-Array суффикса](/sql-reference/aggregate-functions/combinators#-array). С помощью [-Map суффикса](/sql-reference/aggregate-functions/combinators#-map) вы можете применять любую агрегатную функцию к картам. А с помощью [-ForEach суффикса](/sql-reference/aggregate-functions/combinators#-foreach) вы можете применять любую агрегатную функцию к вложенным массивам.

## Источники данных и форматы файлов {#data-sources-and-file-formats}

По сравнению с BigQuery, ClickHouse поддерживает значительно больше форматов файлов и источников данных:

- ClickHouse имеет нативную поддержку загрузки данных в более чем 90 форматах файлов из практически любого источника данных.
- BigQuery поддерживает 5 форматов файлов и 19 источников данных.

## Особенности языка SQL {#sql-language-features}

ClickHouse предоставляет стандартный SQL с множеством расширений и улучшений, которые делают его более удобным для аналитических задач. Например, SQL ClickHouse [поддерживает лямбда-функции](/sql-reference/functions/overview#arrow-operator-and-lambda) и функции высшего порядка, так что вам не нужно распаковывать/разворачивать массивы при применении преобразований. Это большое преимущество по сравнению с другими системами, такими как BigQuery.

## Массивы {#arrays}

По сравнению с 8 функциями массивов в BigQuery, ClickHouse имеет более 80 [встроенных функций работы с массивами](/sql-reference/functions/array-functions) для элегантного и простого моделирования и решения широкого спектра задач.

Типичный шаблон проектирования в ClickHouse — использовать агрегатную функцию [`groupArray`](/sql-reference/aggregate-functions/reference/grouparray), чтобы (временно) преобразовать конкретные значения строк таблицы в массив. Этот массив затем можно удобно обработать с помощью функций массивов, а результат можно преобразовать обратно в отдельные строки таблицы с помощью агрегатной функции [`arrayJoin`](/sql-reference/functions/array-join).

Поскольку SQL ClickHouse поддерживает [функции высшего порядка с лямбда-функциями](/sql-reference/functions/overview#arrow-operator-and-lambda), многие сложные операции с массивами могут быть выполнены простым вызовом одной из встроенных функций высшего порядка, вместо того чтобы временно преобразовывать массивы обратно в таблицы, как это часто [требуется](https://cloud.google.com/bigquery/docs/arrays) в BigQuery, например, для [фильтрации](https://cloud.google.com/bigquery/docs/arrays#filtering_arrays) или [объединения](https://cloud.google.com/bigquery/docs/arrays#zipping_arrays) массивов. В ClickHouse эти операции выполняются простым вызовом функций высшего порядка [`arrayFilter`](/sql-reference/functions/array-functions#arrayfilterfunc-arr1-) и [`arrayZip`](/sql-reference/functions/array-functions#arrayzip) соответственно.

Ниже мы предоставляем сопоставление операций с массивами из BigQuery в ClickHouse:

| BigQuery | ClickHouse |
|----------|------------|
| [ARRAY_CONCAT](https://cloud.google.com/bigquery/docs/reference/standard-sql/array_functions#array_concat) | [arrayConcat](/sql-reference/functions/array-functions#arrayconcat) |
| [ARRAY_LENGTH](https://cloud.google.com/bigquery/docs/reference/standard-sql/array_functions#array_length) | [length](/sql-reference/functions/array-functions#length) |
| [ARRAY_REVERSE](https://cloud.google.com/bigquery/docs/reference/standard-sql/array_functions#array_reverse) | [arrayReverse](/sql-reference/functions/array-functions#arrayreverse) |
| [ARRAY_TO_STRING](https://cloud.google.com/bigquery/docs/reference/standard-sql/array_functions#array_to_string) | [arrayStringConcat](/sql-reference/functions/splitting-merging-functions#arraystringconcat) |
| [GENERATE_ARRAY](https://cloud.google.com/bigquery/docs/reference/standard-sql/array_functions#generate_array) | [range](/sql-reference/functions/array-functions#rangeend-rangestart--end--step) |

**Создать массив с одним элементом для каждой строки в подзапросе**

_Стоит отметить, что следует использовать оператор ARRAY_

```sql
SELECT ARRAY
  (SELECT 1 UNION  ALL
   SELECT 2 UNION ALL
   SELECT 3) AS new_array;

/*-----------*
 | new_array |
 +-----------+
 | [1, 2, 3] |
 *-----------*/
```

_ClickHouse_

[Функция groupArray](/sql-reference/aggregate-functions/reference/grouparray)

```sql
SELECT groupArray(*) AS new_array
FROM
(
    SELECT 1
    UNION ALL
    SELECT 2
    UNION ALL
    SELECT 3
)
   ┌─new_array─┐
1. │ [1,2,3]   │
   └───────────┘
```

**Преобразовать массив в набор строк**

_Стоит отметить, что следует использовать оператор UNNEST_

```sql
SELECT *
FROM UNNEST(['foo', 'bar', 'baz', 'qux', 'corge', 'garply', 'waldo', 'fred'])
  AS element
WITH OFFSET AS offset
ORDER BY offset;

/*----------+--------*
 | element  | offset |
 +----------+--------+
 | foo      | 0      |
 | bar      | 1      |
 | baz      | 2      |
 | qux      | 3      |
 | corge    | 4      |
 | garply   | 5      |
 | waldo    | 6      |
 | fred     | 7      |
 *----------+--------*/
```

_ClickHouse_

[ARRAY JOIN](/sql-reference/statements/select/array-join) clause

```sql
WITH ['foo', 'bar', 'baz', 'qux', 'corge', 'garply', 'waldo', 'fred'] AS values
SELECT element, num-1 AS offset
FROM (SELECT values AS element) AS subquery
ARRAY JOIN element, arrayEnumerate(element) AS num;

/*----------+--------*
 | element  | offset |
 +----------+--------+
 | foo      | 0      |
 | bar      | 1      |
 | baz      | 2      |
 | qux      | 3      |
 | corge    | 4      |
 | garply   | 5      |
 | waldo    | 6      |
 | fred     | 7      |
 *----------+--------*/
```

**Вернуть массив дат**

_Стоит отметить, что следует использовать функцию GENERATE_DATE_ARRAY_

```sql
SELECT GENERATE_DATE_ARRAY('2016-10-05', '2016-10-08') AS example;

/*--------------------------------------------------*
 | example                                          |
 +--------------------------------------------------+
 | [2016-10-05, 2016-10-06, 2016-10-07, 2016-10-08] |
 *--------------------------------------------------*/
```

_С помощью функций range и arrayMap_

_ClickHouse_

```sql
SELECT arrayMap(x -> (toDate('2016-10-05') + x), range(toUInt32((toDate('2016-10-08') - toDate('2016-10-05')) + 1))) AS example

   ┌─example───────────────────────────────────────────────┐
1. │ ['2016-10-05','2016-10-06','2016-10-07','2016-10-08'] │
   └───────────────────────────────────────────────────────┘
```

**Вернуть массив отметок времени**

_Стоит отметить, что следует использовать функцию GENERATE_TIMESTAMP_ARRAY_

```sql
SELECT GENERATE_TIMESTAMP_ARRAY('2016-10-05 00:00:00', '2016-10-07 00:00:00',
                                INTERVAL 1 DAY) AS timestamp_array;

/*--------------------------------------------------------------------------*
 | timestamp_array                                                          |
 +--------------------------------------------------------------------------+
 | [2016-10-05 00:00:00+00, 2016-10-06 00:00:00+00, 2016-10-07 00:00:00+00] |
 *--------------------------------------------------------------------------*/
```

_С помощью функций range и arrayMap_

_ClickHouse_

```sql
SELECT arrayMap(x -> (toDateTime('2016-10-05 00:00:00') + toIntervalDay(x)), range(dateDiff('day', toDateTime('2016-10-05 00:00:00'), toDateTime('2016-10-07 00:00:00')) + 1)) AS timestamp_array

Query id: b324c11f-655b-479f-9337-f4d34fd02190

   ┌─timestamp_array─────────────────────────────────────────────────────┐
1. │ ['2016-10-05 00:00:00','2016-10-06 00:00:00','2016-10-07 00:00:00'] │
   └─────────────────────────────────────────────────────────────────────┘
```

**Фильтрация массивов**

_Стоит отметить, что требует временного преобразования массивов обратно в таблицы с помощью оператора UNNEST_

```sql
WITH Sequences AS
  (SELECT [0, 1, 1, 2, 3, 5] AS some_numbers
   UNION ALL SELECT [2, 4, 8, 16, 32] AS some_numbers
   UNION ALL SELECT [5, 10] AS some_numbers)
SELECT
  ARRAY(SELECT x * 2
        FROM UNNEST(some_numbers) AS x
        WHERE x < 5) AS doubled_less_than_five
FROM Sequences;

/*------------------------*
 | doubled_less_than_five |
 +------------------------+
 | [0, 2, 2, 4, 6]        |
 | [4, 8]                 |
 | []                     |
 *------------------------*/
```

_ClickHouse_

[arrayFilter](/sql-reference/functions/array-functions#arrayfilterfunc-arr1-) function

```sql
WITH Sequences AS
    (
        SELECT [0, 1, 1, 2, 3, 5] AS some_numbers
        UNION ALL
        SELECT [2, 4, 8, 16, 32] AS some_numbers
        UNION ALL
        SELECT [5, 10] AS some_numbers
    )
SELECT arrayMap(x -> (x * 2), arrayFilter(x -> (x < 5), some_numbers)) AS doubled_less_than_five
FROM Sequences;
   ┌─doubled_less_than_five─┐
1. │ [0,2,2,4,6]            │
   └────────────────────────┘
   ┌─doubled_less_than_five─┐
2. │ []                     │
   └────────────────────────┘
   ┌─doubled_less_than_five─┐
3. │ [4,8]                  │
   └────────────────────────┘
```

**Объединение массивов**

_Стоит отметить, что требует временного преобразования массивов обратно в таблицы с помощью оператора UNNEST_

```sql
WITH
  Combinations AS (
    SELECT
      ['a', 'b'] AS letters,
      [1, 2, 3] AS numbers
  )
SELECT
  ARRAY(
    SELECT AS STRUCT
      letters[SAFE_OFFSET(index)] AS letter,
      numbers[SAFE_OFFSET(index)] AS number
    FROM Combinations
    CROSS JOIN
      UNNEST(
        GENERATE_ARRAY(
          0,
          LEAST(ARRAY_LENGTH(letters), ARRAY_LENGTH(numbers)) - 1)) AS index
    ORDER BY index
  );

/*------------------------------*
 | pairs                        |
 +------------------------------+
 | [{ letter: "a", number: 1 }, |
 |  { letter: "b", number: 2 }] |
 *------------------------------*/
```

_ClickHouse_

[arrayZip](/sql-reference/functions/array-functions#arrayzip) function

```sql
WITH Combinations AS
    (
        SELECT
            ['a', 'b'] AS letters,
            [1, 2, 3] AS numbers
    )
SELECT arrayZip(letters, arrayResize(numbers, length(letters))) AS pairs
FROM Combinations;
   ┌─pairs─────────────┐
1. │ [('a',1),('b',2)] │
   └───────────────────┘
```

**Агрегация массивов**

_Стоит отметить, что требует преобразования массивов обратно в таблицы с помощью оператора UNNEST_

```sql
WITH Sequences AS
  (SELECT [0, 1, 1, 2, 3, 5] AS some_numbers
   UNION ALL SELECT [2, 4, 8, 16, 32] AS some_numbers
   UNION ALL SELECT [5, 10] AS some_numbers)
SELECT some_numbers,
  (SELECT SUM(x)
   FROM UNNEST(s.some_numbers) AS x) AS sums
FROM Sequences AS s;

/*--------------------+------*
 | some_numbers       | sums |
 +--------------------+------+
 | [0, 1, 1, 2, 3, 5] | 12   |
 | [2, 4, 8, 16, 32]  | 62   |
 | [5, 10]            | 15   |
 *--------------------+------*/
```

_ClickHouse_

[arraySum](/sql-reference/functions/array-functions#arraysum), [arrayAvg](/sql-reference/functions/array-functions#arrayavg), ... function, или любое из более чем 90 существующих имен агрегатных функций в качестве аргумента для функции [arrayReduce](/sql-reference/functions/array-functions#arrayreduce)

```sql
WITH Sequences AS
    (
        SELECT [0, 1, 1, 2, 3, 5] AS some_numbers
        UNION ALL
        SELECT [2, 4, 8, 16, 32] AS some_numbers
        UNION ALL
        SELECT [5, 10] AS some_numbers
    )
SELECT
    some_numbers,
    arraySum(some_numbers) AS sums
FROM Sequences;
   ┌─some_numbers──┬─sums─┐
1. │ [0,1,1,2,3,5] │   12 │
   └───────────────┴──────┘
   ┌─some_numbers──┬─sums─┐
2. │ [2,4,8,16,32] │   62 │
   └───────────────┴──────┘
   ┌─some_numbers─┬─sums─┐
3. │ [5,10]       │   15 │
   └──────────────┴──────┘
