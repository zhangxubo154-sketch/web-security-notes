# SQL 注入：查询 MySQL / Microsoft 数据库类型和版本

| 项 | 内容 |
|---|---|
| 来源 | PortSwigger Academy |
| 难度 | PRACTITIONER |
| 模块 | SQL 注入 |
| 日期 | 2026-09-18 |
| 耗时 | 待补充 |

## 一、漏洞原理

与 006（Oracle 版本查询）同一类：**从数据库自身读出它的版本**。区别在于**版本查询的写法因数据库而异**：

| 数据库 | 版本查询 |
|---|---|
| Oracle | `SELECT * FROM v$version` |
| Microsoft / MySQL | `SELECT @@version` |
| PostgreSQL | `SELECT version()` |

本题是 MySQL / Microsoft，所以用 `@@version`。

**但这题真正的难点不在版本查询本身，而在注释符 —— 这是我卡住的地方。**

### 卡点：MySQL 的 `--` 后面必须有空格

MySQL 官方文档（*Comments*）写明支持三种注释：

| 写法 | 说明 |
|---|---|
| `# 注释` | 到行尾，**MySQL 独有** |
| `-- 注释` | 到行尾，**但第二个 `-` 后面必须跟至少一个空白字符**（空格 / 制表符 / 换行） |
| `/* 注释 */` | 块注释，可跨行 |

也就是说：**`--` 在 MySQL 上能用，但写成裸 `--`（后面直接跟语句或就是结尾）就不构成注释。**

对比其他数据库：

| 数据库 | `--` 是否需要后面的空格 |
|---|---|
| Oracle | 不需要 |
| PostgreSQL | 不需要 |
| SQL Server | 不需要 |
| **MySQL** | **需要** |

**我一开始一直用裸 `--`，一直失败；换成 `#` 才通过 —— 原因就在这里：** 裸 `--` 没被当成注释，原查询剩余的 `AND ...` 部分没被注掉，SQL 语法就崩了。

### URL 场景下的两个坑

1. **末尾空格会被吞掉** —— 地址栏里写 `-- `（带空格），浏览器可能把空格去掉，到你手里变成裸 `--`。要用 `--+`：`+` 在 query string 里会被解码成**空格**
2. **`#` 是 URL 的片段分隔符** —— 浏览器看到 `#` 后面的内容**根本不会发给服务器**。所以：
   - **Burp Repeater / 原始请求**里 → 可以直接写 `#`
   - **浏览器地址栏**里 → 必须写 `%23`（`#` 的 URL 编码），否则白忙一场

PortSwigger 题解里写的是裸 `#`，因为那是给 **Burp 原始请求**用的。

**稳妥策略**：不确定数据库类型时，用 `-- `（带空格），或者直接 `/* */` —— **块注释两边都不需要空格，四大数据库通吃，最保险**。

### 附带收获：注释符也是数据库指纹

`--` 失败、`#` 成功 → **反面是 MySQL**。所以注释符的差异不只是"要用对"，它和 `@@version` / `v$version` / `version()` 一样，**本身就是判断数据库类型的手段**。

## 二、复现步骤

1. Burp 抓 `category` 参数请求
2. 确认列数与字符串列：

```
category=Gifts' UNION SELECT 'abc','def'#
```
→ 正常返回，确认 **2 列都是字符串类型**

3. 把常量换成版本查询：

```
category=Gifts' UNION SELECT @@version,NULL#
```

4. 页面显示出完整的数据库版本字符串 → 过关

**关键点**：`#` 直接截止到行尾，**不需要空格**，比 MySQL 的 `--` 省事。若坚持用 `--`，必须写成 `-- `（带空格），或 URL 里的 `--+`。

## 三、根本原因

- 用户输入被拼进 SQL 执行（全部 SQL 注入题同源）
- **本题多暴露两点**：
  1. **数据库版本对应用账号可见** —— 版本号能反查已知 CVE，是渗透的指纹识别阶段
  2. **注释符的跨库差异** —— 应用层若做了"过滤 `--`"这种黑名单，MySQL 上换 `#` 或 `/* */` 就能绕过。**黑名单天生打不完**

## 四、修复方案

1. **参数化查询** —— 根治。输入作为值传入，`#`、`--`、`'` 都只是普通字符
2. **不要依赖黑名单过滤注释符** —— 注释写法太多（`#`、`-- `、`/* */`、`/*! */`），堵不完。黑名单只能算纵深防御的一层
3. **关闭详细错误回显** —— 不让报错成为探针
4. **数据库账号最小权限** —— 撤销对版本/元数据视图的访问权

## 五、面试问答

**Q：MySQL 的注释有哪几种？各有什么要求？**
A：`#` 到行尾（MySQL 独有，不需要空格）；`-- ` 到行尾（**第二个减号后必须跟空白字符**，这点和标准 SQL 不同）；`/* */` 块注释。

**Q：为什么 `--` 在 Oracle 能过、在 MySQL 上失败？**
A：MySQL 对 `--` 有额外要求：后面必须跟空格或控制字符。裸 `--` 在 MySQL 不构成注释。

**Q：URL 里写 `#` 会怎样？**
A：`#` 是片段分隔符，浏览器不会把它及之后的内容发给服务器。地址栏里必须用 `%23`；Burp 原始请求里可以写 `#`。

**Q：各数据库怎么查版本？**
A：MySQL / SQL Server `@@version`；Oracle `v$version`；PostgreSQL `version()`。**哪种写法能跑通，就说明对面是什么库。**

**Q：黑名单过滤注释符为什么不可靠？**
A：注释写法多样（`#`、`-- `、`/* */`、MySQL 版本化注释 `/*! */`），且大小写、编码都能变形。根治只能靠参数化查询。

## 六、参考

- [Lab: SQL injection attack, querying the database type and version on MySQL and Microsoft](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft)
- [MySQL 8.0 Reference Manual — Comments](https://dev.mysql.com/doc/refman/8.0/en/comments.html)
- [SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)

## 标签

`#SQL注入` `#MySQL` `#@@version` `#注释符` `#跨库差异`
