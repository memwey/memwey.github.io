+++
title = "在CentOS上使用较新的软件"
date = "2016-08-02T23:30:33+08:00"
updated = "2025-02-15T23:30:33+09:00"
draft = false
description = "借助曾经的 IUS 社区源, 在自带软件老旧的 CentOS 上装上较新版本的 git 与 Python"
[taxonomies]
tags = ["Linux"]
+++

> **更新于 2025-02-15:** [IUS 官方 FAQ](https://ius.io/faq) 已明确说明该项目不再维护, 历史软件包仅保存在 IUS Vault. 本文使用的 CentOS 6 仓库和安装命令均已失效, 请勿在新环境中继续使用; 正文仅作为历史记录保留. 新环境应选择仍受支持的操作系统版本及其官方软件源, 或软件供应方仍在维护的软件源.

公司的服务器都是 `CentOS` 的, 带的软件都比较旧, 让我这个不更新会死星人很难过啊.

开玩笑的, 主要问题是 `git` 版本太旧, `clone` 的时候会报错, 然后也需要 `python3.5`. 在 `git` 上面发现 [ius](https://ius.io/) 来解决这个问题.

[ius](https://ius.io/) 是一个社区项目, 目的是为 [Linux](https://www.kernel.org/) 的企业发行版提供一些更新版本的RPM包.

```bash
sudo yum install epel-release
sudo yum install https://centos6.iuscommunity.org/ius-release.rpm
```

为了解决一些冲突和共存的问题, 有一些 `package` 在 [ius](https://ius.io/) 中的包名有一些改动

```bash
sudo yum install git2u
sudo yum install python35u
```

一般的命名规则是

`{name}{major_version}{minor_version}u`

这样就可以在 `CentOS` 上使用较新的软件啦.
