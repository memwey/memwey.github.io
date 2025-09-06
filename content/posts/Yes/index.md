+++
title = "Yes"
description = "一个有趣的 GNU/Linux 命令"
date = "2018-12-14T23:34:15+08:00"
draft = false
[taxonomies]
tags = ["Linux"]
+++

最近看到一个有趣的 `GNU/Linux` 命令, `yes`

先 `man` 一下

```bash
YES(1)                    BSD General Commands Manual                   YES(1)

NAME
     yes -- be repetitively affirmative

SYNOPSIS
     yes [expletive]

DESCRIPTION
     yes outputs expletive, or, by default, ``y'', forever.

HISTORY
     The yes command appeared in 4.0BSD.

4th Berkeley Distribution        June 6, 1993        4th Berkeley Distribution
(END)
```

emmmmm, 在 `macOS` 下并没有 `--help` 或者 `-h` 的选项

试试 `Linux` 下呢

```bash
➜  / yes --help
Usage: yes [STRING]...
  or:  yes OPTION
Repeatedly output a line with all specified STRING(s), or 'y'.

      --help     display this help and exit
      --version  output version information and exit

GNU coreutils online help: <https://www.gnu.org/software/coreutils/>
Full documentation at: <https://www.gnu.org/software/coreutils/yes>
or available locally via: info '(coreutils) yes invocation'
```

有了

简单来说这个命令就是可以反复的输出一个字符串, 这个字符串的默认值是 `y`

于是有了一些特别的用法, 比如, 注意这里没有 `f`

```bash
yes | rm -r /
```

还有

```bash
yes $(yes yes)
```

好孩子不要乱试

当然这个命令的效果其实一般都有替代, 比如加上 `-f`, `-y` 什么的, 生成一个大文件也可以用 `/dev/urandom` 来做

还可以在冬天做暖手宝, 噗

最后, 跟我一起在命令行输入

```bash
yes 'AMD, yes!'
```

果然人类的本质就是复读机
