


# 作者介绍
*谭峰*: 网名francs，中国开源软件推进联盟PostgreSQL分会特聘专家，《PostgreSQL实战》作者之一，《PostgreSQL 9 Administration Cookbook》译者之一，致力于PostgreSQL技术分享，博客https://postgres.fun
*张文升*:中国开源软件推进联盟PostgreSQL分会核心成员之一，《PostgreSQL实战》作者之一。常年活跃于PostgreSQL、MySQL、Redis等开源技术社区，坚持推动PostgreSQL在中国地区的发展，多次参与组织PostgreSQL全国用户大会。近年来致力于推动PostgreSQL在互联网企业的应用以及企业PostgreSQL培训与技术支持。

# 引言
上篇文章[《PostgreSQL用户应掌握的高级SQL特性》](https://mp.weixin.qq.com/s/Lv_KW8qtbejuY-i69MChpw)介绍了PostgreSQL的典型高级SQL特性，PostgreSQL不仅是关系型数据库，同时支持丰富的NoSQL特性，本篇文章将从《PostgreSQL实战》第9章摘选部分内容介绍PostgreSQL的NoSQL特性，分以下三部分介绍。
- PostgreSQL的JSON和JSONB数据类型简介
- JSON与JSONB读写性能测试
- PostgreSQL全文检索支持JSON和JSONB（PosgreSQL 10 新特性）

# PostgreSQL的JSON和JSONB数据类型
PostgreSQL支持非关系数据类型json(JavaScript Object Notation)，本节介绍json类型、json与jsonb差异、json与jsonb操作符和函数，以及jsonb键值的追加、删除、更新。
## JSON类型简介
PotgreSQL早在9.2版本已经提供了json类型，并且随着大版本的演进，PostgreSQL对json的支持趋于完善，例如提供更多的json函数和操作符方便应用开发，一个简单的json类型例子如下：
```
mydb=> SELECT '{"a":1,"b":2}'::json;
     json      
---------------
 {"a":1,"b":2}
```
为了更好演示json类型，接下来创建一张表，如下所示：
```
mydb=> CREATE TABLE test_json1 (id serial primary key,name json);
CREATE TABLE
```
以上示例定义字段name为json类型，插入表数据，如下所示：
```
mydb=> INSERT INTO test_json1 (name) 
VALUES ('{"col1":1,"col2":"francs","col3":"male"}');
INSERT 0 1

mydb=> INSERT INTO test_json1 (name) 
VALUES ('{"col1":2,"col2":"fp","col3":"female"}');
INSERT 0 1
```
查询表test_json1数据：
```
mydb=> SELECT * FROM test_json1;
 id |                   name                   
----+------------------------------------------
  1 | {"col1":1,"col2":"francs","col3":"male"}
  2 | {"col1":2,"col2":"fp","col3":"female"}
```

## 查询JSON数据
通过->操作符可以查询json数据的键值，如下所示：
```
mydb=> SELECT  name -> 'col2' FROM test_json1 WHERE id=1;
 ?column? 
----------
 "francs"
(1 row)
```
如果想以文本格式返回json字段键值可以使用->>符，如下所示：
```
mydb=> SELECT  name ->> 'col2' FROM test_json1 WHERE id=1;
 ?column? 
----------
 francs
(1 row)
```

## JSONB与JSON差异
PostgreSQL支持两种JSON数据类型：json和jsonb，两种类型在使用上几乎完全相同，两者主要区别为以下：json存储格式为文本而jsonb存储格式为二进制 ，由于存储格式的不同使得两种json数据类型的处理效率不一样，json类型以文本存储并且存储的内容和输入数据一样，当检索json数据时必须重新解析，而jsonb以二进制形式存储已解析好的数据，当检索jsonb数据时不需要重新解析，因此json写入比jsonb快，但检索比jsonb慢，后面会通过测试验证两者读写性能差异。
除了上述介绍的区别之外，json与jsonb在使用过程中还存在差异，例如jsonb输出的键的顺序和输入不一样，如下所示：
```
mydb=> SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::jsonb;
                      jsonb                       
--------------------------------------------------
 {"bar": "baz", "active": false, "balance": 7.77}
(1 row)
```
而json的输出键的顺序和输入完全一样，如下所示：
```
mydb=> SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::json;
                      json                       
-------------------------------------------------
 {"bar": "baz", "balance": 7.77, "active":false}
(1 row)
```
另外，jsonb类型会去掉输入数据中键值的空格，如下所示：
```
mydb=> SELECT ' {"id":1,    "name":"francs"}'::jsonb;
            jsonb            
-----------------------------
 {"id": 1, "name": "francs"}
(1 row)
```
上例中id键与name键输入时是有空格的，输出显示空格键被删除，而json的输出和输入一样，不会删掉空格键：
```
mydb=> SELECT ' {"id":1,    "name":"francs"}'::json;
             json              
-------------------------------
  {"id":1,    "name":"francs"}
(1 row)
```
另外，jsonb会删除重复的键，仅保留最后一个，如下所示：
```
mydb=> SELECT ' {"id":1,                                                              
"name":"francs",
"remark":"a good guy!",
"name":"test"
}'::jsonb;
                       jsonb                        
----------------------------------------------------
 {"id": 1, "name": "test", "remark": "a good guy!"}
(1 row)
```

上面name键重复，仅保留最后一个name键的值，而json数据类型会保留重复的键值。相比json大多数应用场景建议使用jsonb，除非有特殊的需求，比如对json的键顺序有特殊的要求。

## JSONB与JSON操作符
PostgreSQL支持丰富的JSONB和JSON的操作符，举例如下：
以文本格式返回json类型的字段键值可以使用->>符，如下所示：
```
mydb=> SELECT  name ->> 'col2' FROM test_json1 WHERE id=1;
 ?column? 
----------
 francs
(1 row)
```
字符串是否作为顶层键值，如下所示：
```
mydb=> SELECT '{"a":1, "b":2}'::jsonb ? 'a';
 ?column? 
----------
 t
(1 row)
```
删除json数据的键/值，如下所示：
```
mydb=> SELECT '{"a":1, "b":2}'::jsonb - 'a';
 ?column? 
----------
 {"b": 2}
(1 row)
```
## JSONB与JSON函数
json与jsonb相关的函数非常丰富，举例如下：
扩展最外层的json对象成为一组键/值结果集，如下所示：
```
mydb=> SELECT * FROM json_each('{"a":"foo", "b":"bar"}');
 key | value 
-----+-------
 a   | "foo"
 b   | "bar"
(2 rows)
```
以文本形式返回结果，如下所示：
```
mydb=> SELECT * FROM json_each_text('{"a":"foo", "b":"bar"}');
 key | value 
-----+-------
 a   | foo
 b   | bar
(2 rows)
```
一个非常重要的函数为row_to_json()函数，能够将行作为json对象返回，此函数常用来生成json测试数据，比如将一个普通表转换成json类型表：
```
mydb=> SELECT * FROM test_copy WHERE id=1;
 id | name 
----+------
  1 | a
(1 row)

mydb=> SELECT row_to_json(test_copy) FROM test_copy WHERE id=1;
     row_to_json     
---------------------
 {"id":1,"name":"a"}
(1 row)
```
返回最外层的json对像中的键的集合，如下所示：
```
mydb=> SELECT * FROM json_object_keys('{"a":"foo", "b":"bar"}');
 json_object_keys 
------------------
 a
 b
(2 rows)
```

## jsonb键/值的追加、删除、更新
jsonb键/值追加可通过||操作符，如下增加sex键/值：
```
mydb=> SELECT '{"name":"francs","age":"31"}'::jsonb || 
'{"sex":"male"}'::jsonb;
                    ?column?                    
------------------------------------------------
 {"age": "31", "sex": "male", "name": "francs"}
(1 row)
```
jsonb键/值的删除有两种方法，一种是通过操作符号-删除，另一种通过操作符#-删除指定键/值。
通过操作符号-删除键/值如下：
```
mydb=> SELECT '{"name": "James", "email": "james@localhost"}'::jsonb
 - 'email';
     ?column?      
-------------------
 {"name": "James"}
(1 row)

mydb=> SELECT '["red","green","blue"]'::jsonb - 0;
     ?column?      
-------------------
 ["green", "blue"]
```
第二种方法是通过操作符#-删除指定键/值，通常用于有嵌套json数据删除的场景，如下删除嵌套contact中的fax键/值：
```
mydb=> SELECT '{"name": "James", "contact": {"phone": "01234 567890", "fax": "01987 543210"}}'::jsonb #- '{contact,fax}'::text[];
                        ?column?                         
---------------------------------------------------------
 {"name": "James", "contact": {"phone": "01234 567890"}}
(1 row)
```
删除嵌套aliases中的位置为1的键/值，如下所示：
```
mydb=> SELECT '{"name": "James", "aliases": ["Jamie","The Jamester","J Man"]}'::jsonb #- '{aliases,1}'::text[];
                     ?column?                     
--------------------------------------------------
 {"name": "James", "aliases": ["Jamie", "J Man"]}
(1 row)
```
键/值的更新也有两种方式，第一种方式为||操作符，||操作符可以连接json键，也可覆盖重复的键值，如下修改age键的值：
```
mydb=> SELECT '{"name":"francs","age":"31"}'::jsonb || 
'{"age":"32"}'::jsonb;
            ?column?             
---------------------------------
 {"age": "32", "name": "francs"}
(1 row)
```
第二种方式是通过jsonb_set函数，语法如下：
```
jsonb_set(target jsonb, path text[], new_value jsonb[, create_missing boolean])
```
target指源jsonb数据，path指路径，new_value指更新后的键值，create_missing 值为 true表示如果键不存在则添加，create_missing 值为 false表示如果键不存在则不添加，示例如下：
```
mydb=> SELECT jsonb_set('{"name":"francs","age":"31"}'::jsonb,'{age}','"32"'::jsonb,false);
            jsonb_set            
---------------------------------
 {"age": "32", "name": "francs"}
(1 row)

mydb=> SELECT jsonb_set('{"name":"francs","age":"31"}'::jsonb,'{sex}','"male"'::jsonb,true);
                   jsonb_set                    
------------------------------------------------
 {"age": "31", "sex": "male", "name": "francs"}
(1 row)
```

## 给JSONB类型创建索引
这一小节介绍给jsonb数据类型创建索引，jsonb数据类型支持GIN索引，为了便于说明，假如一个json字段内容如下，并且以jsonb格式存储。
```
{
  "id": 1, 
  "user_id": 1440933, 
  "user_name": "1_francs", 
  "create_time": "2017-08-03 16:22:05.528432+08"
}
```
假如存储以上jsonb数据的字段名为user_info，表名为tbl_user_jsonb，在user_info字段上创建GIN索引语法如下：
```
CREATE INDEX idx_gin ON tbl_user_jsonb USING gin(user_info);
```
jsonb上的GIN索引支持@>、?、 ?&、?|操作符，例如以下查询将会使用索引。
```
SELECT * FROM tbl_user_jsonb WHERE user_info @> '{"user_name": "1_frans"}'
```
但是以下基于jsonb键值的查询不会走索引idx_gin，如下所示：
```
SELECT * FROM tbl_user_jsonb WHERE user_info->>'user_name'= '1_francs';
```
如果要想提升基于jsonb类型的键值检索效率，可以在jsonb数据类型对应的键值上创建索引，如下所示：
```
CREATE INDEX idx_gin_user_infob_user_name ON tbl_user_jsonb USING btree 
((user_info ->> 'user_name'));
```
创建以上索引后，上述根据user_info->>'user_name'键值查询的SQL将会走索引。

# JSON与JSONB读写性能测试
上一小节介绍了jsonb数据类型索引创建相关内容，本小节将对json、jsonb读写性能进行简单对比，在第3章数据类型章节中介绍json、jsonb数据类型时提到了两者读写性能的差异，主要表现为json写入时比jsonb快，但检索时比jsonb慢，主要原因为：json存储格式为文本而jsonb存储格式为二进制，存储格式的不同使得两种json数据类型的处理效率不一样，json类型存储的内容和输入数据一样，当检索json数据时必须重新解析，而jsonb以二进制形式存储已解析好的数据，当检索jsonb数据时不需要重新解析。 

## 构建JSON、JSONB测试表
- tbl_user_json:： json 数据类型表，200万数据；
- tbl_user_jsonb： jsonb 数据类型表，200万数据；

首先创建user_ini表并插入200万测试数据，如下：
```
mydb=> CREATE TABLE user_ini(id int4 ,user_id int8, user_name character 
varying(64),create_time timestamp(6) with time zone default 
clock_timestamp());
CREATE TABLE

mydb=> INSERT INTO user_ini(id,user_id,user_name)
SELECT r,round(random()*2000000), r || '_francs' 
FROM generate_series(1,2000000) as r;
INSERT 0 2000000
```
计划使用user_ini表数据生成json、jsonb数据，创建user_ini_json、user_ini_jsonb表，如下所示：
```
mydb=> CREATE TABLE tbl_user_json(id serial, user_info json);
CREATE TABLE
mydb=> CREATE TABLE tbl_user_jsonb(id serial, user_info jsonb);
CREATE TABLE
```

## JSON与JSONB表写性能测试
根据user_ini数据通过row_to_json函数向表user_ini_json插入200万json数据，如下：
```
mydb=> \timing
Timing is on.

mydb=> INSERT INTO tbl_user_json(user_info) SELECT row_to_json(user_ini)
FROM user_ini;
INSERT 0 2000000
Time: 13825.974 ms (00:13.826)
```
从以上结果看出tbl_user_json插入200万数据花了13秒左右；接着根据user_ini表数据生成200万jsonb数据并插入表tbl_user_jsonb，如下：
```
mydb=> INSERT INTO tbl_user_jsonb(user_info)
 SELECT row_to_json(user_ini)::jsonb FROM user_ini;    
INSERT 0 2000000
Time: 20756.993 ms (00:20.757)
```
从以上看出tbl_user_jsonb表插入200万jsonb数据花了20秒左右，正好验证了json数据写入比jsonb快，比较两表占用空间大小，如下所示：
```
mydb=> \dt+ tbl_user_json
                       List of relations
 Schema |     Name      | Type  | Owner  |  Size  | Description 
--------+---------------+-------+--------+--------+-------------
 pguser | tbl_user_json | table | pguser | 281 MB | 
(1 row)

mydb=> \dt+ tbl_user_jsonb
                        List of relations
 Schema |      Name      | Type  | Owner  |  Size  | Description 
--------+----------------+-------+--------+--------+-------------
 pguser | tbl_user_jsonb | table | pguser | 333 MB | 
(1 row)
```
从占用空间来看，同样的数据量jsonb数据类型占用空间比json稍大。
查询tbl_user_json表的一条测试数据，如下：
```
mydb=> SELECT * FROM tbl_user_json LIMIT 1;
   id    |                                             user_info                                             
---------+------------------------------------------------------------------------------------
 2000001 | {"id":1,"user_id":1182883,"user_name":"1_francs","create_time":"2017-08-03T20:59:27.42741+08:00"}
(1 row)
```
## JSON与JSONB表读性能测试
对于json、jsonb读性能测试我们选择基于json、jsonb键值查询的场景，例如，根据user_info字段的user_name键的值查询，如下所示：
```
mydb=> EXPLAIN ANALYZE SELECT * FROM tbl_user_jsonb WHERE user_info->>'user_name'='1_francs';
                          QUERY PLAN                                                     
-------------------------------------------------------------------------------------
 Seq Scan on tbl_user_jsonb  (cost=0.00..72859.90 rows=10042 width=143) (actual time=0.023..524.843 rows=1 loops=1)
   Filter: ((user_info ->> 'user_name'::text) = '1_francs'::text)
   Rows Removed by Filter: 1999999
 Planning time: 0.091 ms
 Execution time: 524.876 ms
(5 rows)
```
上述SQL执行时间为524毫秒左右，基于user_info字段的user_name键值创建btree索引如下：
```
mydb=> CREATE INDEX idx_jsonb ON tbl_user_jsonb USING btree((user_info->>'user_name'));
```
再次执行上述查询，如下所示：
```
mydb=> EXPLAIN ANALYZE SELECT * FROM tbl_user_jsonb WHERE user_info->>'user_name'='1_francs';
                                   QUERY PLAN                                                         
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on tbl_user_jsonb  (cost=155.93..14113.93 rows=10000 width=143) (actual time=0.027..0.027 rows=1 loops=1)
   Recheck Cond: ((user_info ->> 'user_name'::text) = '1_francs'::text)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_jsonb  (cost=0.00..153.43 rows=10000 width=0) (actual time=0.021..0.021 rows=1 loops=1)
         Index Cond: ((user_info ->> 'user_name'::text) = '1_francs'::text)
 Planning time: 0.091 ms
 Execution time: 0.060 ms
(7 rows)
```
根据上述执行计划看出走了索引，并且SQL时间下降到0.060ms。为更好的对比tbl_user_json、tbl_user_jsonb表基于键值查询的效率，计划根据user_info字段id键进行范围扫描对比性能，创建索引如下：
```
mydb=> CREATE INDEX idx_gin_user_info_id ON tbl_user_json USING btree 
(((user_info ->> 'id')::integer));
CREATE INDEX

mydb=> CREATE INDEX idx_gin_user_infob_id ON tbl_user_jsonb USING btree 
(((user_info ->> 'id')::integer));
CREATE INDEX
```
索引创建后，查询tbl_user_json表如下：
```
mydb=> EXPLAIN ANALYZE SELECT id,user_info->'id',user_info->'user_name'
 FROM tbl_user_json 
WHERE (user_info->>'id')::int4>1 AND (user_info->>'id')::int4<10000;
                               QUERY PLAN                                                            
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on tbl_user_json  (cost=166.30..14178.17 rows=10329 width=68) (actual time=1.167..26.534 rows=9998 loops=1)
   Recheck Cond: ((((user_info ->> 'id'::text))::integer > 1) AND (((user_info ->> 'id'::text))::integer < 10000))
   Heap Blocks: exact=338
   ->  Bitmap Index Scan on idx_gin_user_info_id  (cost=0.00..163.72 rows=10329 width=0) (actual time=1.110..1.110 rows=19996 loops=
1)
         Index Cond: ((((user_info ->> 'id'::text))::integer > 1) AND (((user_info ->> 'id'::text))::integer < 10000))
 Planning time: 0.094 ms
 Execution time: 27.092 ms
(7 rows)
```

根据以上看出，查询表tbl_user_json的user_info字段id键值在1到10000范围内的记录走了索引，并且执行时间为27.092毫秒，接着测试tbl_user_jsonb表同样SQL的检索性能，如下所示：
```
mydb=> EXPLAIN ANALYZE SELECT id,user_info->'id',user_info->'user_name'
 FROM tbl_user_jsonb 
WHERE (user_info->>'id')::int4>1 AND (user_info->>'id')::int4<10000;
                        QUERY PLAN                                                           
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on tbl_user_jsonb  (cost=158.93..14316.93 rows=10000 width=68) (actual time=1.140..8.116 rows=9998 loops=1)
   Recheck Cond: ((((user_info ->> 'id'::text))::integer > 1) AND (((user_info ->> 'id'::text))::integer < 10000))
   Heap Blocks: exact=393
   ->  Bitmap Index Scan on idx_gin_user_infob_id  (cost=0.00..156.43 rows=10000 width=0) (actual time=1.058..1.058 rows=18992 loops
=1)
         Index Cond: ((((user_info ->> 'id'::text))::integer > 1) AND (((user_info ->> 'id'::text))::integer < 10000))
 Planning time: 0.104 ms
 Execution time: 8.656 ms
(7 rows)
```
根据以上看出，查询表tbl_user_jsonb的user_info字段id键值在1到10000范围内的记录走了索引并且执行时间为8.656毫秒，从这个测试看出jsonb检索比json效率高。
从以上两个测试看出，正好验证了“json写入比jsonb快，但检索时比jsonb慢”的观点，值得一提的是如果需要通过key/value进行检索，例如以下:
```
SELECT * FROM tbl_user_jsonb WHERE user_info @> '{"user_name": "2_francs"}';
```
这时执行计划为全表扫描，如下所示：
```
mydb=> EXPLAIN ANALYZE SELECT * FROM tbl_user_jsonb WHERE user_info @> '{"user_name": "2_francs"}';
                                QUERY PLAN                                                     
------------------------------------------------------------------------------------
 Seq Scan on tbl_user_jsonb  (cost=0.00..67733.00 rows=2000 width=143) (actual time=0.018..582.207 rows=1 loops=1)
   Filter: (user_info @> '{"user_name": "2_francs"}'::jsonb)
   Rows Removed by Filter: 1999999
 Planning time: 0.065 ms
 Execution time: 582.232 ms
(5 rows)
```
从以上看出执行时间为582毫秒左右，在tbl_user_jsonb字段user_info上创建gin索引，如下所示：
```
mydb=> CREATE INDEX idx_tbl_user_jsonb_user_Info ON tbl_user_jsonb USING gin
 (user_Info);
CREATE INDEX
```
索引创建后，再次执行以下，如下所示：
```
mydb=> EXPLAIN ANALYZE SELECT * FROM tbl_user_jsonb WHERE user_info @> '{"user_name": "2_francs"}';
                                  QUERY PLAN                                                           
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on tbl_user_jsonb  (cost=37.50..3554.34 rows=2000 width=143) (actual time=0.079..0.080 rows=1 loops=1)
   Recheck Cond: (user_info @> '{"user_name": "2_francs"}'::jsonb)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_tbl_user_jsonb_user_info  (cost=0.00..37.00 rows=2000 width=0) (actual time=0.069..0.069 rows=1 loop
s=1)
         Index Cond: (user_info @> '{"user_name": "2_francs"}'::jsonb)
 Planning time: 0.094 ms
 Execution time: 0.114 ms
(7 rows)
```
从以上看出走了索引，并且执行时间下降到了0.114毫秒。
这一小节测试了json、jsonb数据类型读写性能差异，验证了json写入时比jsonb快，但检索时比jsonb慢的观点。
# PostgreSQ全文检索支持JSON和JSONB
这一小节将介绍PostgreSQL10的一个新特性：全文检索支持json、jsonb数据类型，本小节分两部分，第一部分简单介绍PostgreSQL全文检索，第二部分演示全文检索对json、jsonb数据类型的支持。

## PostgreSQL全文检索简介
对于大多数应用全文检索很少放到数据库中实现，一般使用单独的全文检索引擎，例如基于SQL全文检索引擎Sphinx。PostgreSQL支持全文检索，对于规模不大的应用如果不想搭建专门的搜索引擎，PostgreSQL的全文检索也可以满足需求。
如果没有使用专门的搜索引擎，大部检索需要通过数据库like操作匹配，这种检索方式主要缺点在于：
- 不能很好的支持索引，通常需全表扫描检索数据，数据量大时检索性能很低。
- 不提供检索结果排序，当输出结果数据量非常大时表现更加明显。
PostgreSQL全文检索能有效地解决这个问题，PostgreSQL全文检索通过以下两种数据类型来实现。

### tsvector
tsvector全文检索数据类型代表一个被优化的可以基于搜索的文档，将一串字符串转换成tsvector全文检索数据类型，如下：
```
mydb=> SELECT 'Hello,cat,how are u? cat is smiling! '::tsvector;
                     tsvector                     
--------------------------------------------------
 'Hello,cat,how' 'are' 'cat' 'is' 'smiling!' 'u?'
(1 row)
```
可以看到，字符串的内容被分隔成好几段，但通过::tsvector只是做类型转换，没有进行数据标准化处理，对于英文全文检索可通过函数to_tsvector进行数据标准化，如下所示：
```
mydb=> SELECT to_tsvector('english','Hello cat,');
    to_tsvector    
-------------------
 'cat':2 'hello':1
(1 row)
```

### tsquery
tsquery表示一个文本查询，存储用于搜索的词，并且支持布尔操作&、|、!，将字符串转换成tsquery，如下所示：
```
mydb=> SELECT  'hello&cat'::tsquery;
     tsquery     
-----------------
 'hello' & 'cat'
(1 row)
```
上述只是转换成tsquery类型，而并没有做标准化，使用to_tsquery函数可以执行标准化，如下所示：
```
mydb=> SELECT to_tsquery( 'hello&cat' );
   to_tsquery    
-----------------
 'hello' & 'cat'
(1 row)
```
一个全文检索示例如下，检索字符串是否包括hello和cat字符，本例中返回真。
```
mydb=> SELECT to_tsvector('english','Hello cat,how are u') @@ 
to_tsquery( 'hello&cat' );
 ?column? 
----------
 t
(1 row)
```
检索字符串是否包含字符hello和dog，本例中返回假。
```
mydb=> SELECT to_tsvector('english','Hello cat,how are u') @@
 to_tsquery( 'hello&dog' );
 ?column? 
----------
 f
(1 row)
```
有兴趣的读者可以测试tsquery的其他操作符，例如|、!等。

注意：这里使用了带双参数的to_tsvector函数，函数to_tsvector双参数的格式如下：
to_tsvector([ config regconfig , ] document text)。本节to_tsvector函数指定了config参数为english，如果不指定config参数，则默认使用default_text_search_config参数的配置。

### 英文全文检索例子
下面演示一个英文全文检索示例，创建一张测试表并插入200万测试数据，如下所示：
```
mydb=> CREATE TABLE test_search(id int4,name text);
CREATE TABLE
mydb=> INSERT INTO test_search(id,name) SELECT n, n||'_francs'
FROM generate_series(1,2000000) n;
INSERT 0 2000000
```
执行以下SQL，查询test_search表name字段包含字符1_francs的记录。
```
mydb=> SELECT * FROM test_search WHERE name LIKE '1_francs';
 id |   name   
----+----------
  1 | 1_francs
(1 row)
```
执行计划如下：
```
mydb=> EXPLAIN ANALYZE SELECT * FROM test_search WHERE name LIKE '1_francs';
                                      QUERY PLAN                                                  
-------------------------------------------------------------------------------------Seq Scan on test_search  (cost=0.00..38465.04 rows=204 width=18) (actual time=0.022..261.766 rows=1 loops=1)
   Filter: (name ~~ '1_francs'::text)
   Rows Removed by Filter: 1999999
 Planning time: 0.101 ms
 Execution time: 261.796 ms
(5 rows)
```
以上执行计划走了全表扫描，执行时间为261毫秒左右，性能很低，接着创建索引，如下所示：
```
mydb=> CREATE INDEX idx_gin_search ON test_search USING gin 
(to_tsvector('english',name));
CREATE INDEX
```
执行以下SQL，查询test_search表name字段包含字符1_francs的记录。
```

```
再次查看执行计划和执行时间，如下所示：
```
mydb=> EXPLAIN ANALYZE SELECT * FROM test_search WHERE to_tsvector('english',name) @@ 
to_tsquery('english','1_francs');
                                        QUERY PLAN                                                        
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on test_search  (cost=18.39..128.38 rows=50 width=36) (actual time=0.071..0.071 rows=1 loops=1)
   Recheck Cond: (to_tsvector('english'::regconfig, name) @@ '''1'' & ''franc'''::tsquery)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_gin_search  (cost=0.00..18.38 rows=50 width=0) (actual time=0.064..0.064 rows=1 loops=1)
         Index Cond: (to_tsvector('english'::regconfig, name) @@ '''1'' & ''franc'''::tsquery)
 Planning time: 0.122 ms
 Execution time: 0.104 ms
(7 rows)
```
创建索引后，以上查询走了索引并且执行时间下降到0.104毫秒，性能提升了3个数量级，值得一提的是如果SQL改成以下，则不走索引，如下所示：
```
mydb=> EXPLAIN ANALYZE SELECT * FROM test_search
 WHERE to_tsvector(name) @@ to_tsquery('1_francs');
                                 QUERY PLAN                                                    
-------------------------------------------------------------------------------------
 Seq Scan on test_search  (cost=0.00..1037730.00 rows=50 width=18) (actual time=0.036..10297.764 rows=1 loops=1)
   Filter: (to_tsvector(name) @@ to_tsquery('1_francs'::text))
   Rows Removed by Filter: 1999999
 Planning time: 0.098 ms
 Execution time: 10297.787 ms
(5 rows)
```
由于创建索引时使用的是to_tsvector('english',name)函数索引，带了两个参数，因此where条件中的to_tsvector函数带两个参数才能走索引，而to_tsvector(name)不走索引。

## JSON、JSONB全文检索实践
在PostgreSQL10版本之前全文检索不支持json和jsonb数据类型，10版本的一个重要特性是全文检索支持json和jsonb数据类型，这一小节演示下10版本的这个新特性。

### PostgreSQL 10版本与9.6版本to_tsvector函数的差异
先来看下9.6版本to_tsvector函数，如下：
```
[postgres@pghost1 ~]$ psql francs francs
psql (9.6.3)
Type "help" for help.

mydb=> \df *to_tsvector*
                                List of functions
   Schema   |       Name        | Result data type | Argument data types |  Type  
------------+-------------------+------------------+---------------------+--------
 pg_catalog | array_to_tsvector | tsvector         | text[]              | normal
 pg_catalog | to_tsvector       | tsvector         | regconfig, text     | normal
 pg_catalog | to_tsvector       | tsvector         | text                | normal
(3 rows)
```
从以上看出9.6版本to_tsvector函数的输入参数仅支持text、text[]数据类型，接着看下10版本的to_tsvector函数，如下所示：
```
[postgres@pghost1 ~]$ psql mydb pguser
psql (10.0)
Type "help" for help.
mydb=> \df *to_tsvector*
                                List of functions
   Schema   |       Name        | Result data type | Argument data types |  Type  
------------+-------------------+------------------+---------------------+--------
 pg_catalog | array_to_tsvector | tsvector         | text[]              | normal
 pg_catalog | to_tsvector       | tsvector         | json                | normal
 pg_catalog | to_tsvector       | tsvector         | jsonb               | normal
 pg_catalog | to_tsvector       | tsvector         | regconfig, json     | normal
 pg_catalog | to_tsvector       | tsvector         | regconfig, jsonb    | normal
 pg_catalog | to_tsvector       | tsvector         | regconfig, text     | normal
 pg_catalog | to_tsvector       | tsvector         | text                | normal
(7 rows)
```
从以上看出，10版本的to_tsvector函数支持的数据类型增加了json和jsonb。

### 创建数据生成函数
为了便于生成测试数据，创建以下两个函数用来随机生成指定长度的字符串，创建random_range(int4, int4)函数如下：
```
CREATE OR REPLACE FUNCTION random_range(int4, int4)
RETURNS int4
LANGUAGE SQL
AS $$
    SELECT ($1 + FLOOR(($2 - $1 + 1) * random() ))::int4;
$$;
```
接着创建random_text_simple(length int4)函数，此函数会调用random_range(int4, int4)函数。
```
CREATE OR REPLACE FUNCTION random_text_simple(length int4)
RETURNS text
LANGUAGE PLPGSQL
AS $$
DECLARE
    possible_chars text := '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ';
    output text := '';
    i int4;
    pos int4;
BEGIN

    FOR i IN 1..length LOOP
        pos := random_range(1, length(possible_chars));
        output := output || substr(possible_chars, pos, 1);
    END LOOP;

    RETURN output;
END;
$$;
```
random_text_simple(length int4)函数可以随机生成指定长度字符串，如下随机生成含三位字符的字符串。
```
mydb=> SELECT random_text_simple(3);
 random_text_simple 
--------------------
 LL9
(1 row)
```
随机生成含六位字符的字符串，如下所示：
```
mydb=> SELECT random_text_simple(6);
 random_text_simple 
--------------------
 B81BPW
(1 row)
```
后面会用到这个函数生成测试数据。

### 创建JSON测试表
创建user_ini测试表，并通过random_text_simple(length int4)函数插入100万随机生成六位字符的字符串测试数据，如下所示：
```
mydb=> CREATE TABLE user_ini(id int4 ,user_id int8,
user_name character varying(64),
create_time timestamp(6) with time zone default clock_timestamp());
CREATE TABLE

mydb=> INSERT INTO user_ini(id,user_id,user_name) 
SELECT r,round(random()*1000000), random_text_simple(6) 
FROM generate_series(1,1000000) as r;
INSERT 0 1000000
```
创建tbl_user_search_json表，并通过row_to_json函数将表user_ini行数据转换成json数据，如下所示：
```
mydb=> CREATE TABLE tbl_user_search_json(id serial, user_info json);
CREATE TABLE

mydb=> INSERT INTO tbl_user_search_json(user_info)
 SELECT row_to_json(user_ini) FROM user_ini;
INSERT 0 1000000
```
生成的数据如下：
```
mydb=> SELECT * FROM tbl_user_search_json LIMIT 1;
 id |                                            user_info                                            
----+-----------------------------------------------------------------------------------------------
  1 | {"id":1,"user_id":186536,"user_name":"KTU89H","create_time":"2017-08-05T15:59:25.359148+08:00"}
(1 row)
```

### JSON数据全文检索测试
使用全文检索查询表tbl_user_search_json的user_info字段中包含KTU89H字符的记录，如下所示：
```
mydb=> SELECT * FROM tbl_user_search_json 
WHERE to_tsvector('english',user_info) @@ to_tsquery('ENGLISH','KTU89H');
 id |                                            user_info                                            
----+----------------------------------------------------------------------------------------
  1 | {"id":1,"user_id":186536,"user_name":"KTU89H","create_time":"2017-08-05T15:59:25.359148+08:00"}
(1 row)
```
以上SQL能正常执行说明全文检索支持json数据类型，只是上述SQL走了全表扫描性能低，执行时间为8061毫秒，如下所示：
```
mydb=> EXPLAIN ANALYZE SELECT * FROM tbl_user_search_json
 WHERE to_tsvector('english',user_info) @@ to_tsquery('ENGLISH','KTU89H');
                                        QUERY PLAN                                                         
-----------------------------------------------------------------------------------
 Seq Scan on tbl_user_search_json  (cost=0.00..279513.00 rows=5000 width=104) (actual time=0.046..8061.858 rows=1 loops=1)
   Filter: (to_tsvector('english'::regconfig, user_info) @@ '''ktu89h'''::tsquery)
   Rows Removed by Filter: 999999
 Planning time: 0.091 ms
 Execution time: 8061.880 ms
(5 rows)
```
创建如下索引：
```
mydb=>  CREATE INDEX idx_gin_search_json ON tbl_user_search_json USING 
gin(to_tsvector('english',user_info));
 CREATE INDEX
```
索引创建后，再次执行以下SQL，如下所示：
```
mydb=> EXPLAIN ANALYZE SELECT * FROM tbl_user_search_json WHERE to_tsvector('english',user_info) @@ to_tsquery('ENGLISH','KTU89H');
                                        QUERY PLAN                                                           
-------------------------------------------------------------------------------------
 Bitmap Heap Scan on tbl_user_search_json  (cost=50.75..7876.06 rows=5000 width=104) (actual time=0.024..0.024 rows=1 loops=1)
   Recheck Cond: (to_tsvector('english'::regconfig, user_info) @@ '''ktu89h'''::tsquery)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on idx_gin_search_json  (cost=0.00..49.50 rows=5000 width=0) (actual time=0.018..0.018 rows=1 loops=1)
         Index Cond: (to_tsvector('english'::regconfig, user_info) @@ '''ktu89h'''::tsquery)
 Planning time: 0.113 ms
 Execution time: 0.057 ms
(7 rows)
```
从上述执行计划看出走了索引，并且执行时间降为0.057毫秒，性能非常不错。
这一小节前一部分对PostgreSQL全文检索的实现做了简单介绍，并且给出了一个英文检索的例子，后一部分通过示例介绍了PostgreSQL10的一个新特性，即全文检索支持json、jsonb类型。

# 总结
本文介绍了PostgreSQL的NoSQL特性，首先介绍了json和jsonb数据类型，之后通过示例对比json、jsonb数据类型读写性能差异，最后介绍了PostgreSQL全文检索对json、jsonb类型的支持（PostgreSQL10新特性）；值得一提的是，PostgreSQL对中文全文检索也是支持的，有兴趣的读者可自行测试。

# 《PostgreSQL实战》书籍推荐
*作者:*       谭峰、张文升
*出版日期:*   2018年7月
*页数:*       415页
*定价:*       89元
*本书特色*
中国开源软件推进联盟PostgreSQL分会特聘专家撰写，国内多位开源数据库专家鼎力推荐。
基于PostgreSQL 10 编写，重点介绍SQL高级特性、并行查询、分区表、物理复制、逻辑复制、备份恢复、高可用、性能优化、PostGIS等，涵盖大量实战用例。
*购买链接:*    京东:   https://item.jd.com/12405774.html

*PostgreSQL*中文社区欢迎广大技术人员投稿，投稿邮箱：*press@postgres.cn*


