1、`DISTINCT` 关键词用于去除重复记录，该关键词应用于所有列

2、`REGEXP` 是用于查询时进行正则匹配的，默认是不区分大小写的，可以使用 `REGEXP BINARY` 的形式进行大小写区分。
```sql
select product_name from products where product_name REGEXP "1000 | 2000";
-- 1000 | 2000, 这里表示 OR 的关系 

select product_name from products where product_name REGEXP "[123]Ton";
-- [123] 表示匹配1或2或3，例如：1Ton, 所以 [] 是另一种形式的 OR
-- [^123] 而这种形式就是，不是1或者不是2或者不是3，表示前面那种相反的意思


-- 范围匹配：
-- 例如匹配数字，按照之前的概念我们可以写成：[0123456789]
-- 但是为了简便可以写成：[0-9]，范围不一定是要全部，可以是一部分的形式：[1-3]、[6-9]
-- 甚至可以说字母：[a-z]、[A-Z]
select product_name from products where product_name REGEXP "[1-5]Ton";


-- 匹配特殊字符：
-- 所谓特殊字符就是正则的语法规则对应的符号，我们必须进行转义才能进行正常匹配，例如：
-- 匹配点号(.)，为了匹配特殊字符，必须用 \\ 为前导
select product_name from products where product_name REGEXP "\\.";


-- 匹配字符类：
-- [:alnum:]    任意字母和数字（同[a-zA-Z0-9]）
-- [:alpha:]    任意字符（同[a-zA-Z]）
-- [:blank:]    空格和制表（同[\\t]）
-- [:cntrl:]    ASCII控制字符（ASCII 0到31和127）
-- [:digit:]    任意数字（同[0-9]）
-- [:graph:]    与[:print:]相同，但不包括空格
-- [:lower:]    任意小写字母（同[a-z]）
-- [:print:]    任意可打印字符
-- [:punct:]    既不在[:alnum:]又不在[:cntrl:]中的任意字符
-- [:space:]    包括空格在内的任意空白字符（同[\\f\\n\\r\\t\\v]）
-- [:upper:]    任意大写字母（同[A-Z]）
-- [:xdigit:]   任意十六进制数字（同[a-fA-F0-9]）


-- 匹配多个实例（一般指前面的选项连续出现多少次才会匹配）
-- *        0个或多个匹配
-- +        1个或多个匹配（等于{1,}）
-- ?        0个或1个匹配（等于{0,1}）
-- {n}      指定数目的匹配
-- {n,}     不少于指定数目的匹配
-- {n,m}    匹配数目的范围（m不超过255）
-- [[:digit:]]{4} 表示匹配数字连续四位的


-- 特定位置的匹配
-- ^ 文本的开始（以什么开头），注意: ^在[]中时表示否定这个集合
-- $ 文本的结尾（以什么结尾）
-- [[:<:]] 词的开始
-- [[:>:]] 词的结尾
```

3、`Concat()` 函数拼接两个列的值, 该函数用来拼接多个串，要拼接的串之间用逗号分隔
```sql
SELECT concat(username, '(', id, ')') from user;

+--------------------------------+
| concat(username, '(', id, ')') |
+--------------------------------+
| user01(1)                      |
| user02(2)                      |
| user03(3)                      |
| user04(4)                      |
| user05(5)                      |
+--------------------------------+
5 rows in set (0.00 sec)
```

4、删除数据右侧多余的空格
```sql
SELECT concat("Stard", Trim(" And "), "End") as 'Trim', concat("Stard", LTrim(" And "), "End") as 'LTrim', concat("Stard", RTrim(" And "), "End") as 'RTrim';

+-------------+--------------+--------------+
| Trim        | LTrim        | RTrim        |
+-------------+--------------+--------------+
| StardAndEnd | StardAnd End | Stard AndEnd |
+-------------+--------------+--------------+

-- Trim: 移除前后空格
-- LTrim: 移除左边空格
-- RTrim: 移除右边空格
-- 提示: as 可以指定查询结果集对应列标题别名
```