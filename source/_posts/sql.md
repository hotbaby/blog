---
title: sql
date: 2017-12-01
tags: SQL
toc: true
comments: true
---

MySQL数据库远程访问

```sql
GRANT ALL PRIVILEGES ON *.* TO 'USERNAME'@'%' IDENTIFIED BY 'PASSWORD' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

创建数据库

```sql
CREATE DATABASE database_name CHARACTER SET utf8;
```

导出数据库表

```shell
mysqldump -h127.0.0.1 -uusername -ppassword
database_name table_name > database_name.table_name.sql;

mysqldump -h127.0.0.1 -uusername -ppassword
database_name table_name --where="id>100" > database_name.table_name.sql;
```

导出数据库表结构

```shell
mysqldump  -hhost -P3306 -uusername -ppassword  database_name --no-data > db_name_schema.sql
```

导入数据库

```shell
mysql -h127.0.0.1 -uusername -ppassword database_name < database_name.table_name.sql;
mysql -uusername -ppassword; 
> source sql_file_path;
```

数据从一张表导出到另一个张表

```sql
INSERT INTO database_name.table_name
            (field1,
             field2)
SELECT field1,
       field2
FROM   database_name.table_name;
```

根据一张表更新另一张表

```sql
UPDATE kdreader.library_book
INNER JOIN store_book
set        kdreader.library_book.author_id=store_book.author_id
WHERE      kdreader.library_book.name=store_book.titl;
```

修改表结构：增加Column

```sql
ALTER TABLE table_name_example ADD COLUMN field_name field_type DEFAULT default_value;
```

修改表结构：更新Column

```sql
ALTER TABLE table_name_example ALTER COLUMN field_name field_type DEFAULT default_value;
```

修改表结构：删除Column

```sql
ALTER TABLE table_name_example DROP COLUMN field_name;
```

修改表结构：删除unique约束

```sql
ALTER TABLE table_name_example DROP INDEX unique_key_name;
```

表重命名

```sql
ALTER TABLE old_name RENAME TO new_name;
```

增加Unique约束

```sql
ALTER TABLE Persons ADD CONSTRAINT uni_PersonID UNIQUE (Id_P,LastName);
```

分组查询

```sql
SELECT book_labels.label_id,
       Count(book_labels.book_id) cnt
FROM   store_book_labels AS book_labels
GROUP  BY book_labels.label_id
ORDER  BY cnt DESC
LIMIT  20;
```

## Reference

* [W3School SQL](http://www.w3school.com.cn/sql/index.asp)
* [developer mysqldump](https://dev.mysql.com/doc/refman/5.7/en/mysqldump.html)
* [MySQL foreign key](http://www.mysqltutorial.org/mysql-foreign-key/)

