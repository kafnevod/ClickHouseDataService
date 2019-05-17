## Цель

Создание ClickHouse DataService (ICSSoft.STORMNET.Business.ClickHouseDataService) в рамках проекта
[Flexberry ORM](https://github.com/Flexberry/NewPlatform.Flexberry.ORM).

## Функциональные требования

DataService должен обеспечивать объектно-реляционное отображение запросов из Microsoft .NET Framework
в SQL-запросы с базе данных ClickHouse.

## Особенности реализации

ClickHouse обеспечиает эффективную работу с "большими" таблицами - сотни миллионов, миллиарды и более записей.
В связи с этим большое внимание необходимо уделить качественному формированию SQL-запроса,
обеспечивающее быстрое и эффективное (с минимальными промежуточными таблицами) выполнение запросов.

Особенности модели данных ClickHouse:

- ClickHouse не поддерживает запросы `UPDATE`, `DELETE`
В связи с этим необходимо реализовать только операции выборки (`SELECT`) данных.

- Как правило ClickHouse работает с одной "широкой" (возможно денормализованной) таблицей размером сотни миллионов и более записей. ClickHouse - поколоночная база.
Время выполнения запроса прямо пропорционально числу выбиремых столбцов.
В связи с этим необходимо минимизировать число выбираемых столбцов в запросе `SELECT`

- Операции `JOIN` "большой" таблицы, как правило, производятся с небольщими (до миллиона записей) справочными таблицами.
В связи с этим необходимо:

  * В начальном запросе и в подзапросах `LEFT JOIN` использовать не сами соединяемые таблицы, а выборки из них 
  `SELECT FROM column, column FROM table ...`
 
  * Минимизировать число выбираемых столбцов в кажом запросе `SELECT`

  * Начальный запрос `SELECT` должен формироваться для "большой" таблицы  и использовать оператор `IN` 
  для выборки записей из большой таблицы  с последующим соединением выбранных записей с требуемыми спавочными таблицами.

- В качестве справочных таблиц может быть использованы справочники ClickHouse, представлящие собой синхронизируемые копии справочных таблиц других баз данных (Postgres, MySQ:, MSSQL, Oracle, ...). В этом случае в качнестве UUID'ов используются поля типа Int64. Необходимо поддержать возможность использования такого типа ссылочных полей.
 
 ## Формирование запроса SELET к "большой" таблице
 
 ## Формирование запроса SELET ... JOIN ... к "большой" таблице
 
 Рассмотрим аггрегацию трех классов:
 ![Пример аггрегации трех классов](/images/joinClassExample.png)
 
Рассмотрим запрос выборки полей таблицы `BigTable`, в которых поле `colimnA` таблицы `dictC` содержит слово 'Perm'.
В резулттате необходимо вывести поля:
- BigTable.columnA;
- dicktA.columnC;
- dictB.columnA;
- dictB.columnB.

Стандартно сформированный запрос выглядит следующим образом:
```
select
 "BigTable"."columnA",
 "dicktA"."columnC",
 "dictB"."columnA",
 "dictB"."columnB"
from (
 select
  "BigTable"."columnA",
  "BigTable"."linkDictA"
 from "BigTable"
) as "BigTable"

join (
 select
  "dictA"."primaryKey",
  "dictA"."columnC",
  "dictA"."linkDictB"
 from "dictA"
) as "dictA" ON "BigTable"."linkDictA" = "dictA"."primaryKey"

join (
 select
  "DictB"."primaryKey",
  "DictB"."columnA",
  "DictB"."columnB"
  from "DictB"
) as "DictB" ON "dictA"."linkDictB" = "dictB"."primaryKey"
WHERE "DictB"."columnA" = 'Perm'
```

> Обратите внимание, что в подзапросах вместо полных таблиц  необходимо использовать выборки минимально необходимых столбцов `SELECT` соединяемых таблиц.

Данные запросы в ClickHouse и Postgres выполняются достаточно долго, так как в соединении принимают участи 
все записи `BigTable`.

Для оптимизаии времени выполнения необходимо 
- сначала выбрать оператором `IN` необходимые строки и столбцы `BigTable`
- выбранную подтаблицу соединить со справочниками.

Оптимизированный запрос должен выглядеть так:
```
select
 "BigTable"."columnA",
 "dicktA"."columnC",
 "dictB"."columnA",
 "dictB"."columnB"
from (
 select
  "BigTable"."columnA",
  "BigTable"."linkDictA"
 from "BigTable"

 WHERE "BigTable"."linkDictA" IN (
  SELECT "dictA"."primaryKey"
  FROM (
   SELECT
    "DictB"."primaryKey",
    "DictB"."columnA",
   FROM "DictB" WHERE "DictB"."columnA" = 'Perm'
  ) AS "DictB"
  JOIN {
   SELECT
    "dictA"."primaryKey",
    "dictA"."linkDictB"
  ) AS "dictA" ON "dictA"."linkDictB" = "DictB"."primaryKey"

) as "BigTable"

join (
 select
  "dictA"."primaryKey",
  "dictA"."columnC",
  "dictA"."linkDictB"
 from "dictA"
) as "dictA" ON "BigTable"."linkDictA" = "dictA"."primaryKey"

join (
 select
  "DictB"."primaryKey",
  "DictB"."columnA",
  "DictB"."columnB"
) as "DictB" ON "dictA"."linkDictB" = "dictB"."primaryKey"
```
Операторы подзапроса `IN` выделены большими буквами.
данные оператиры обеспечивают формирование одностолбцовой таблицы, содержащей 
идентификаторы строк таблиц словаря `DictA` по которым необходимо сделать выборку 
строк и столбцов таблицы `BigTable`.

Выбранные строки и столбцы как и в исходном примере соединяются с требуемыми строками и столбцами справочников
`DictA`, `DictB`.

