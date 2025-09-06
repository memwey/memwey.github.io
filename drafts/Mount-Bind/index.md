+++
title = "Mount Bind"
date = "2019-10-08T23:50:02+08:00"
draft = true
description = ""
[taxonomies]
tags = ["Linux"]
+++

## 问题来源

默认情况下,Docker会把东西装在 `/var/lib/docker` 目录下,当然这个可以通过修改 `/etc/docker/daemon.json` 中的值来修改.

Docker在把各种乱七八糟的东西都放在
