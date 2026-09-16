# SQL 注入导致显示隐藏数据

| 项 | 内容 |
|---|---|
| 来源 | PortSwigger Academy |
| 难度 | APPRENTICE |
| 模块 | SQL 注入 |
| 日期 | 待填写 |
| 耗时 | 待填写 |

> 这是一份**待填写**的骨架。做完题后把每一节补上，不要把这一段留着。

## 一、漏洞原理

待填写。提示：这道题的查询是

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

想清楚："未上架商品"是被**哪个条件**挡住的？如果那个条件失效会发生什么？为什么它会失效？

## 二、复现步骤

待填写。至少写清楚：

1. 在哪个功能点、哪个参数上动手
2. 用的 payload 是什么
3. 响应发生了什么变化（前后对比）

## 三、根本原因

待填写。从"用户输入被直接拼进 SQL"这个层面往下说。

## 四、修复方案

待填写。提示方向：参数化查询（预编译）为什么能根治它？只做转义/黑名单为什么不够？

## 五、面试问答

待填写。可以先自己回答这两个：

- 什么是 SQL 注入？为什么它能读到本不该读的数据？
- 参数化查询是怎么防住的？和字符串拼接的本质区别在哪？

## 六、参考

- [Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)

## 标签

`#SQL注入` `#WHERE子句` `#入门`
