下面是数据相关管理操作：
```sql
-- 查看所有的数据库
show databases;

-- 创建数据库
CREATE DATABASES test;

-- 进入数据库
use test
-- Database changed

-- 查看当前进入的数据库
SELECT DATABASE();

-- 删除数据
DROP DATABASE test;

-- 查看当前数据库下说有的表
show tables;

-- 创建表
CREATE TABLE <表名> (字段名 字段类型(长度), ...);

-- 查看表结构
DESCRIBE <表名>;

-- 插入一条记录（插入值是所有字段时，前面的字段名列表可以省略）
INSERT INTO <表名> [(字段名, ...)] VALUES (字段对应的值, ...)

-- 查询表
SELECT <字段名 | *> FROM <表名> WHERE <condition>;
```