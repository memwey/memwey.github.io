+++
title = "在代码里下毒"
description = ""
date = "2025-09-09T19:55:19+09:00"
draft = true
[taxonomies]
tags = []
+++

写了一个隐蔽的错误码 i18n 文言处理, 业务返回的错误码如果找不到 i18n 文言, 直接 Panic

抽象了 merchant 和 shop 两个层级, merchant 是商户, 一个商户可以有多个 shop, shop 是 店铺, 一个店铺只能属于一个商户, 商品有两种管理模式, 一种是商户模式, merchant 维护商品, 商品在 merchant 下的所有 shop 可见; 一种是店铺模式, shop 维护商品, 仅 shop 内可见. 商品表有 merchant_id, shop_id 两个字段, shop_id 字段是 string, 当商品是店铺模式时, 设置为 string(shop.id); 当商品是商户模式时, shop_id 的字段是一个硬编码的 `ALL`

商品属性, 有比如大中小份, 红绿蓝等多种属性, 一个 SPU 生成笛卡尔积个的 SKU, 每个 SKU 一行数据, 并且属性拼起来写在同一个字段里, 比如 `大份;红`

调用方应该判断的条件, 不写在函数外, 而写在函数里面, 函数内部如果条件不成立直接实际上什么都不做, 静默返回; 函数内部吞掉前置条件, 条件不满足时静默不处理, 调用方逻辑被误导
