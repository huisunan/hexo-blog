---
title: 批量修改字段字符集和表表字符集，sql生成
date: 2024-04-11 09:24
tags: mysql
categories: 
---

<!--more-->

表字符集修改

```sql

SELECT
    CONCAT(
        'ALTER TABLE ',
        TABLE_NAME,
        ' CONVERT TO CHARACTER SET utf8mb4;'
    )
FROM
    information_schema. TABLES
WHERE
    TABLE_SCHEMA = 'dataBaseName';

```

表字段字符集修改

```sql

SELECT
    CONCAT(
        'ALTER TABLE `',
        TABLE_NAME,
        '` MODIFY `',
        COLUMN_NAME,
        '` ',
        DATA_TYPE,
        '(',
        CHARACTER_MAXIMUM_LENGTH,
        ')',

        ' CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci',
         if(COLUMN_DEFAULT is null ,'',concat(' default \'',COLUMN_DEFAULT,'\'')),
        (
            CASE
            WHEN IS_NULLABLE = 'NO' THEN
                ' NOT NULL'
            ELSE
                ''
            END
        ),
        ';'
    )
FROM
    information_schema.COLUMNS
WHERE
    TABLE_SCHEMA = 'table_name'
AND (DATA_TYPE = 'varchar' OR DATA_TYPE = 'char')
and TABLE_NAME not in ('flyway_schema_history','undo_log');
```