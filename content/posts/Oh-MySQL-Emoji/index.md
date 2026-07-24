+++
title = "Oh MySQL Emoji"
date = "2019-10-14T11:12:33+08:00"
draft = false
description = "MySQL 中的 Emoji 到 Unicode / UTF-8 的编码漫谈"
[taxonomies]
tags = ["Database", "MySQL"]
+++

又是关于 `MySQL` 的, 标题来自于 <轮到你了> 这部~~烂尾~~日剧, 里面非常魔性的 Oh~my~Ju~lia~

在MySQL里面如何保存 [Emoji](https://en.wikipedia.org/wiki/Emoji), 这个问题搜一搜很容易找到答案, 设置 **CHARSET** 为 `utf8mb4`, 看来天下苦 `MySQL` 久矣.

## 基础知识

计算机最早是美国人发明的, 那时的编码都是 `ASCII` 码, 毕竟英文字符加上标点符号也就那么多嘛, 一个字节绰绰有余.

但是随着计算机的发展, 计算机面向的国家和语言越来越多, 一个字节根本不够用了, 于是就有了很多面向特定语言的编码, 比如汉字的`GBK`, `Big5`, 韩文的 `EUC-KR`, 日文的 `Shift_JIS` 什么的. 总体思想是, 既然一个字节不够用, 那就多几个字节就是了嘛.

但是问题又出现了, 同一个编码在不同语言中会有不同的含义,比如韩文编码 `EUC-KR` 中 `한국어` 的编码值正好对应着汉字编码 `GBK` 中的 `茄惫绢`. 还有那个著名的 `瓣B变巨肚`, 也正是同样的原因. 这样仿佛就变成了巴别塔的故事. 人们虽然用着同样的二进制编码, 但是却表达着不同的意思; 就像人们虽然同样发出声带的震动, 但是却无法互相理解对方的话语.

[国际标准化组织](https://en.wikipedia.org/wiki/International_Organization_for_Standardization) 和 [统一码联盟](https://en.wikipedia.org/wiki/Unicode_Consortium) 意识到, 这样下去是不行的, 不如搞一个超大的字符集, 然后把人类所有字符都弄进去, 这样人类都可以用同一个标准了. 于是, 他们分别制定了 [USC](https://en.wikipedia.org/wiki/Universal_Coded_Character_Set) 和 [Unicode](https://en.wikipedia.org/wiki/Unicode). 当然, 如果两个标准各搞各的, 那就和他们最初的想法背道而驰了, 所以目前这两个字符集在实际使用中是一致的. 一般还是把这个字符集叫 `Unicode`.

`Unicode` 的范围上限是 `0x10FFFF`, 换算成十进制就是 `1,114,111` 这么多. 所以如果以后外星人的文字太多, 说不定这个范围就不够了.

其实这里悄悄的偷换了一个概念, 之前在说编码, 现在变成了字符集了. 在很多之前提到的编码中, 这些编码既是字符集, 又是直接的编码. 而在 `Unicode` 中, 字符集实际上是与编码分开的两个概念, 而对应的编码, 实际上就是我们经常见到的 `UTF`. 其中 `UTF-8` 是我们最常见的一种编码了.

`UTF-8` 最大的特点, 就是它是一种变长的编码方式, 而且完全兼容 `ASCII`码.

`UTF-8` 的编码规则也很简单，只有二条:

* 对于单字节的符号, 字节的第一位设为`0`, 后面7位为这个符号的 Unicode 码. 对于 `0 - 127` 位的字符，UTF-8 编码和 ASCII 码是完全相同的.

* 对于 `n` 字节的符号 `(n > 1)` , 第一个字节的前 `n` 位都设为 `1`, 第 `n + 1` 位设为 `0`, 后面字节的前两位一律设为 `10`. 剩下的没有提及的二进制位, 全部为这个符号的 Unicode 码.

| Unicode符号范围     |        UTF-8编码方式
| --- | --- |
| U+0000 - U+007F | 0xxxxxxx |
| U+0080 - U+07FF | 110xxxxx 10xxxxxx |
| U+0800 - U+FFFF | 1110xxxx 10xxxxxx 10xxxxxx |
| U+010000 - U+10FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx |

在这里我们发现, `U+0000 - U+FFFF` 这个**平面**(Plane), 即**基本多文种平面**, 简称 `BMP`, 用 `UTF-8` 进行编码, 只需要三个字节.

## 轮到你了

在 `MySQL` 的 **CHARSET** 中, `utf8` 支持存储 1 - 3字节的字符, 即对应 `Unicode` 中 `BMP` 的部分, 而 [Emoji](https://en.wikipedia.org/wiki/Emoji), 则大多数在 `BMP` 之外. 所以 `MySQL` 中的 `utf8` 是假的, 是化学的成分, 是加了特技的. 如果想储存真正的 `UTF-8` 的内容, 就一定要使用 `utf8mb4`.

当然, 并不是所有的 [Emoji](https://en.wikipedia.org/wiki/Emoji) 都在 `BMP` 之外, 比如 ☺ 这个表情, 它的编码是 `U+263A`, 还有 ☹️ , 编码是 `U+2639`.

当然, 也不是所有的汉字都被包含在了 `BMP` 里面了, 在 [这里](https://www.compart.com/en/unicode/block) 可以看到字符是按照一定的规律划分为 `Block` 放进来的, 在**表意文字补充平面**也就是 `U+20000 - U+2FFFF` 这个**平面**(Plane), 还是有很多 `CJK` 命名的 `Block` 的.

## 再来一瓶

你以为坑到这里就结束了吗

```SQL
CREATE TABLE `emoji_test` (
  `id` INT UNSIGNED NOT NULL AUTO_INCREMENT,
  `emoji` VARCHAR(1024) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

```SQL
SET NAMES utf8mb4;
```

```bash
MySQL [test]> SELECT * FROM emoji_test;
+----+-------+
| id | emoji |
+----+-------+
|  8 | 😃      |
|  9 | 😂      |
| 10 | 🤦      |
+----+-------+
3 rows in set (0.001 sec)

MySQL [test]> SELECT * FROM emoji_test WHERE emoji = '😂';
+----+-------+
| id | emoji |
+----+-------+
|  8 | 😃      |
|  9 | 😂      |
| 10 | 🤦      |
+----+-------+
3 rows in set (0.001 sec)
```

是不是很神奇呢, 看下表的信息呢

```SQL
MySQL [test]> SHOW TABLE STATUS WHERE Name = 'emoji_test';
+------------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
| Name       | Engine | Version | Row_format | Rows | Avg_row_length | Data_length | Max_data_length | Index_length | Data_free | Auto_increment | Create_time         | Update_time         | Check_time | Collation          | Checksum | Create_options | Comment |
+------------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
| emoji_test | InnoDB |      10 | Dynamic    |    3 |           5461 |       16384 |               0 |            0 |         0 |             11 | 2019-10-16 11:16:28 | 2019-10-16 11:25:10 | NULL       | utf8mb4_general_ci |     NULL |                |         |
+------------+--------+---------+------------+------+----------------+-------------+-----------------+--------------+-----------+----------------+---------------------+---------------------+------------+--------------------+----------+----------------+---------+
1 row in set (0.001 sec)
```

可以看到此时默认的字符序是 `utf8mb4_general_ci`, 使用 `WEIGHT_STRING` 来查询这些 Emoji 的字符序

```SQL
MySQL [test]> SET @s = '😂' COLLATE utf8mb4_general_ci;
Query OK, 0 rows affected (0.003 sec)

MySQL [test]> SELECT @s, HEX(@s), HEX(WEIGHT_STRING(@s));
+------+----------+------------------------+
| @s   | HEX(@s)  | HEX(WEIGHT_STRING(@s)) |
+------+----------+------------------------+
| 😂     | F09F9882 | FFFD                   |
+------+----------+------------------------+
1 row in set (0.001 sec)

MySQL [test]> SET @s = '😃' COLLATE utf8mb4_general_ci;
Query OK, 0 rows affected (0.001 sec)

MySQL [test]> SELECT @s, HEX(@s), HEX(WEIGHT_STRING(@s));
+------+----------+------------------------+
| @s   | HEX(@s)  | HEX(WEIGHT_STRING(@s)) |
+------+----------+------------------------+
| 😃     | F09F9883 | FFFD                   |
+------+----------+------------------------+
1 row in set (0.001 sec)
```

可以看到, 它们的编码虽然不同, 但是字符序的值是相同的, 都是 `0xFFFD`, 所以在匹配的时候, 它们被认为是同一个字符, 这又是 `MySQL` 的一个大坑

要是想粗暴点解决问题的话, 直接使用 `utf8mb4_bin` 作为字符序就好了

## 又来一瓶

但是这还不是结束, 当业务代码连接数据库并插入一些 Emoji 时, 还是可能遇到如下错误

```
ERROR 1366: Incorrect string value: '\xF0\x9D\x8C\x86' for column 'column_name' at row 1
```

这个是因为客户端到 MySQL 服务器的连接还是 utf8 而非 utf8mb4, 所以你需要在业务逻辑中做类似如下语句的事情

```
SET NAMES utf8mb4
```

或者修改 MySQL 的配置文件, 当然这样不太现实, 需要重启 MySQL 进程. 

这样你才能终于用上 Emoji

## 参考资料

1. [How can I search by emoji in MySQL using utf8mb4?](https://stackoverflow.com/questions/41147829/how-can-i-search-by-emoji-in-mysql-using-utf8mb4)
2. [字符编码笔记：ASCII，Unicode 和 UTF-8
](http://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)
3. [如何通俗地理解Unicode、UTF-8、ASCII、GBK等字符编码？](https://www.zhihu.com/question/60072951)
4. [In MySQL, never use “utf8”. Use “utf8mb4”.](https://medium.com/@adamhooper/in-mysql-never-use-utf8-use-utf8mb4-11761243e434)
5. [How to support full Unicode in MySQL databases](https://mathiasbynens.be/notes/mysql-utf8mb4)
6. [Connection Character Sets and Collations](https://dev.mysql.com/doc/refman/5.7/en/charset-connection.html)
7. [“Incorrect string value” when trying to insert UTF-8 into MySQL via JDBC?](https://stackoverflow.com/questions/10957238/incorrect-string-value-when-trying-to-insert-utf-8-into-mysql-via-jdbc)

## 扩展阅读

1. [其实你并不懂 Unicode](https://zhuanlan.zhihu.com/p/53714077)
2. [Dark corners of Unicode](https://eev.ee/blog/2015/09/12/dark-corners-of-unicode/)
3. [MySQL · 实现分析 · 对字符集和字符序支持的实现](http://mysql.taobao.org/monthly/2017/03/06/)
4. [unicodedata — Unicode Database](https://docs.python.org/3.7/library/unicodedata.html)
