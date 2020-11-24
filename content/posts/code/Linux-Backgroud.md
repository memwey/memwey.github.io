---
title: "Linux后台命令"
date: 2016-06-15T23:28:24+08:00
draft: false
toc: false
images:
tags: 
  - Linux
---

最近在跑 DHT 网络拟真的时候涉及到一些 linux 后台运行的一些东西,现在总结一下

例如
```bash
nohup ../src/OverSim -c ChordChurn -u Cmdenv > out.file 2>&1 &
```
使得 `../src/OverSim -c ChordChurn -u Cmdenv` 命令在后台运行,并将 `stderr` 重定向到 `stdout` 中然后再将 `stdout` 重定向到 `out.file`, 并且在当前终端关闭时仍然运行

具体说一下吧

## nohup
退出终端的时候,一般在终端中运行的进程会随之关闭,因为此时终端中的子进程会收到 `SIGHUP` 信号. 而 `nohup` 使得进程忽略所有 `SIGHUP` 信号. `stdout` 默认会重定向到 `nohup.out` 文件中

## &
在命令中末尾加入 `&` 符号,使进程在后台运行

## 2>&1
放在 `>` 后面的 `&`, 表示重定向的目标不是一个文件, 而是一个文件描述符, 文件描述符

* `1 => stdout`
* `2 => stderr`
* `0 => stdin`

## 其他一些命令和操作

`Control+z` 可以发出 `SIGSTOP` 信号, 使当前前台进程暂停并放入后台

`fg [job_id]`使进程在前台运行

`bg [job_id]`使进程在后台运行

`jobs` 查看后台进程
