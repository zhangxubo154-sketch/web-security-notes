# SQL 注入 UNION 攻击

| 项 | 内容 |
|---|---|
| 来源 | PortSwigger Academy |
| 难度 | APPRENTICE |
| 模块 | SQL 注入 |
| 日期 | 待填写 |
| 耗时 | 待填写 |

> 这是一份**待填写**的骨架。做完题后把每一节补上，不要把这一段留着。

## 一、漏洞原理

待填写。提示：UNION 是 SQL 里**合并两个查询结果**的操作符。想清楚：

- UNION **不是**让你读新数据，它让你把**另一条你自己构造的查询**的结果拼到原结果后面 —— 那你怎么才能借它读到别的表？
- UNION 要求前后两个查询**列数一致**，所以第一件事是什么？
- 为什么 UNION 注入比上一题（WHERE 子句）更危险？上一题只能改条件，这一题能得到什么？

## 二、复现步骤

待填写。**这道题是分步的，每一步都写下来**：

1. **判断列数** —— 用什么方法？你用了哪种（`ORDER BY` 递增 / `UNION SELECT NULL` 递增）？为什么第一次试的数会失败？
2. **判断回显位** —— 列数对了以后，怎么知道哪一列的内容会显示在页面上？
3. **取数据** —— 拿到回显位之后，你查了什么？

每一步都要写清楚：

- 用的 payload 是什么
- 页面**前后有什么变化**（不回显 / 报错 / 显示出来）

**关键点**：这一题的核心不在"打出来"，而在**"怎么一步步试出该填几个 NULL"**。把这个推理过程写清楚，比 payload 本身值钱。

## 三、根本原因

待填写。提示：和上一题一样，根因是用户输入被拼进 SQL。但这里多问一层 ——

**为什么攻击者能写进 `UNION SELECT`？** 仅仅是"没过滤"吗，还是"数据库把用户输入当成了 SQL 代码而不是数据"？这两个说法差别在哪？

## 四、修复方案

待填写。提示方向：

- 参数化查询为什么能同时防住这一题和上一题？
- 为什么"只过滤 UNION 关键字"是错的？（想想大小写、注释、编码绕过）

## 五、面试问答

待填写。可以先自己回答这几个：

- UNION 注入和普通注入（改 WHERE 条件）的区别是什么？
- 怎么判断一个注入点是几列？有几种方法，各有什么优缺点？
- 什么是"回显位"？如果页面**完全不回显**，UNION 注入还有用吗，那时候该换什么思路？

## 六、参考

- [Lab: SQL injection UNION attack, determining the number of columns returned by the query](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns)
- [SQL injection UNION attacks（知识点页）](https://portswigger.net/web-security/sql-injection/union-attacks)

## 标签

`#SQL注入` `#UNION` `#判断列数` `#回显位`
