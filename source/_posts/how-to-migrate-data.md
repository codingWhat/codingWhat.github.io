---
title: 如何优雅迁移数据
date: 2022-10-23 18:25:12
tags:
---

## 数据迁移的背景
基本上数据迁移无外乎如下原因
- 业务增长
- 降本增效

常见迁移的策略有如下两种方式
1. 停机迁移
2. 不停机迁移

像金融场景会采用第一种，一般会提前发布公告，在迁移期间服务不可用。  
而大部分互联网的场景更倾向于采用不停机的优雅迁移策略。


## 迁移时机
- 已在测试环境模拟完毕
- 凌晨, 数据量少

## 迁移阶段
- 全量拷贝(mysqldump/XtraBackup, redis-shake)
- 写旧、读旧
- 先写旧再写新，读旧(异步diff检测)
- 先写新再写旧，读新(异步diff检测)
- 写新、读新

## 全量拷贝

## 校准和修复逻辑


## 实战
```sql
CREATE  TABLE  tbl_old (
    id int(11) not null auto_increment primary key,
    data varchar(20) not null,
    created_at timestamp not null default current_timestamp,
    updated_at timestamp not null default  current_timestamp on update
)

```