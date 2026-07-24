+++
title = "Oh MySQL IN Subquery"
date = "2019-09-21T15:22:58+08:00"
draft = false
description = ""
[taxonomies]
tags = ["Database", "MySQL"]
+++

感觉 `MySQL` 的坑实在是有点多,记录一下这个 `MySQL` 的坑, 也记录一下这个教训吧, 下次在数据库中直接操作一定要多小心

## 现象回放
现在有两张表, 表结构如下, 无关字段已经省略

* team
```
+-------------+---------------------+
| Field       | Type                |
+-------------+---------------------+
| id          | int(10) unsigned    |
| status      | tinyint(3) unsigned |
+-------------+---------------------+
```

* player
```
+-------------+---------------------+
| Field       | Type                |
+-------------+---------------------+
| id          | int(10) unsigned    |
| team_id     | int(10) unsigned    |
| status      | tinyint(3) unsigned |
+-------------+---------------------+
```

容易理解, 这是简单的一对多的关系, 一个足球队 team 里面, 有多个球员 player

现在想取出有 player 的状态为 `1` 的 team

```SQL
SELECT * FROM team WHERE id IN (SELECT id FROM (SELECT team_id FROM player WHERE status = 1) AS a);
```

理论上, 这条语句是不能执行的, 注意这里

>  ...... SELECT id FROM (SELECT team_id FROM ......

但是, 不知道为何, 这条语句是可以执行的, 而且等价于

```SQL
SELECT * FROM team;
```

如果单独把最外层的 IN 里面的 subquery 取出来, `MySQL` 会报错

```SQL
SELECT id FROM (SELECT team_id FROM player WHERE status = 1) AS a;
```

> ERROR 1054 (42S22): Unknown column 'id' in 'field list'

再试着把 id 改为不存在的字段

```SQL
SELECT * FROM team WHERE id IN (SELECT not_exist_field FROM (SELECT team_id FROM player WHERE status = 1) AS a);
```

> ERROR 1054 (42S22): Unknown column 'not_exist_field' in 'field list'

这样才能如预期的报错

## 问题排查

先 EXPLAIN 试试呢
```SQL
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: team
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 2
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: player
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 5
     filtered: 20.00
        Extra: Using where; FirstMatch(team); Using join buffer (Block Nested Loop)
2 rows in set, 2 warnings (0.001 sec)
```

好像看不出来什么问题呢, 不过有 warnings, 看一下呢

```SQL
SHOW WARNINGS\G
```
```
*************************** 1. row ***************************
  Level: Note
   Code: 1276
Message: Field or reference 'test.team.id' of SELECT #2 was resolved in SELECT #1
*************************** 2. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `test`.`team`.`id` AS `id`,`test`.`team`.`status` AS `status` from `test`.`team` semi join (`test`.`player`) where (`test`.`player`.`status` = 1)
2 rows in set (0.001 sec)
```

这里出现了一个 `semi join`, 没有见过呢, 查看一下文档, 大意就是, 比如使用 `INNER JOIN` 的时候, 会返回匹配次数个结果. 但是我们并不关注匹配的次数, 比如如下语句, 我想取出有球员的 status 为 0 的球队, 可以这样写

```SQL
SELECT * FROM team INNER JOIN player ON team.id = team_id WHERE player.status = 0;
```

如果一个球队里有多个 status 为 0 的球员, 那么就会出现多个记录, 比如这样

```
+----+--------+----+---------+--------+
| id | status | id | team_id | status |
+----+--------+----+---------+--------+
|  1 |      0 |  1 |       1 |      0 |
|  1 |      0 |  2 |       1 |      0 |
|  2 |      0 |  5 |       2 |      0 |
|  2 |      0 |  6 |       2 |      0 |
+----+--------+----+---------+--------+
```

这样明显有些冗余的数据了. 当然我们可以用 `DISTINCT` 什么的再处理一遍, 但是这样效率会比较低. 那么, 就可以用类似的子查询就方便多了

```SQL
SELECT * FROM team WHERE id IN (SELECT team_id FROM player WHERE status = 0);
```

返回的结果也简洁多了

```
+----+--------+
| id | status |
+----+--------+
|  1 |      0 |
|  2 |      0 |
+----+--------+
```

当然, 要这样优化还是有很多条件的, 林林总总的, 可以去官方文档查看

看了这么多, 感觉还是和这个问题没什么关系啊, 试着 EXPLAIN 一下正确的语句

```SQL
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: team
   partitions: NULL
         type: ALL
possible_keys: PRIMARY
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 2
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: player
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 5
     filtered: 20.00
        Extra: Using where; FirstMatch(team); Using join buffer (Block Nested Loop)
2 rows in set, 1 warning (0.001 sec)
```

再看看 warnings

```SQL
*************************** 1. row ***************************
  Level: Note
   Code: 1003
Message: /* select#1 */ select `test`.`team`.`id` AS `id`,`test`.`team`.`status` AS `status` from `test`.`team` semi join (`test`.`player`) where ((`test`.`player`.`team_id` = `test`.`team`.`id`) and (`test`.`player`.`status` = 1))
1 row in set (0.001 sec)
```

对照着实际运行的语句的 Note, 发现错误的语句缺少了以下这个条件

> ((`test`.`player`.`team_id` = `test`.`team`.`id`)

难道就是你! 但是为什么又有

> Field or reference 'test.team.id' of SELECT #2 was resolved in SELECT #1

这个问题呢

## 破案

后来翻了翻文档, 发现这压根不是 `MySQL` 的 Bug, 而是 SQL 一条相当坑的作用域规则, 子查询里解析不了的列名, 会跑到外层查询去找.

关键还是在那个派生表 `a`

```SQL
(SELECT team_id FROM player WHERE status = 1) AS a
```

它只有 `team_id` 一列, 根本没有 `id`. 于是外面那句 `SELECT id FROM a` 里的 `id`, 在 `a` 里是找不到的. 按照作用域规则, 内层找不到就往外层找, 而外层的 `team` 表恰好有 `id`, 所以这个 `id` 就被解析成了 `team.id`. 这也正是那条 Note 想告诉我们的

> Field or reference 'test.team.id' of SELECT #2 was resolved in SELECT #1

`id` 变成 `team.id` 之后, 子查询就成了「对当前这一行 `team`, 返回它自己的 `team.id`, `a` 里有几行就返回几次」, 整个条件相当于

```SQL
team.id IN (SELECT team.id FROM a)
```

只要 `a` 非空, 也就是全表里存在任意一个 `status` 为 `1` 的球员, 这个 `IN` 对每一行 `team` 都恒为真. 本该起作用的关联条件 `player.team_id = team.id` 从头到尾就没参与进来, 所以每支球队都被捞了出来, 等价于 `SELECT * FROM team`. 再对照前面两条 Note 里 `semi join` 的 `where`, 一个有 `team_id = team.id`, 一个没有, 也就说得通了.

回过头看之前那几个奇怪的现象, 也都顺理成章了

* 单独跑 `SELECT id FROM a` 会报错, 是因为没有外层可以借, `id` 真的无处可寻
* 嵌套里把 `id` 换成 `not_exist_field` 也报错, 是因为这个名字在 `a` 和外层 `team` 里都不存在
* 唯独 `id` 不报错, 只是因为它恰好是外层 `team` 的列, 被悄悄借走了, 报错被吃掉, 结果却全错

所以并不是 `MySQL` 抽风, 而是「外层恰好有个同名列」, 把一句本该报错的语句, 变成了能跑但结果完全不对的语句, 属实有点阴险.

教训还是开头那句, 在数据库里直接操作一定要多小心. 具体到这个坑, 一个好习惯是子查询里的列名都带上表名或别名, 写 `a.team_id` 而不是光秃秃的 `id`, 一旦写错会立刻报错, 而不会被外层悄悄借走. 另外, 这个规则在 `DELETE FROM team WHERE id IN (SELECT id FROM ...)` 里最要命, 列名一写错解析成外层的 `id`, 条件恒真, 整张表就没了.

## 参考资料

1. [Optimizing Subqueries, Derived Tables, and View References with Semijoin Transformations](https://dev.mysql.com/doc/refman/5.7/en/semijoins.html)
2. [Correlated Subqueries](https://dev.mysql.com/doc/refman/8.0/en/correlated-subqueries.html)
