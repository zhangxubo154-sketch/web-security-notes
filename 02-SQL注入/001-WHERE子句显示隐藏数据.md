# SQL 注入：利用 WHERE 子句检索隐藏数据

| 项 | 内容 |
|---|---|
| 来源 | PortSwigger Academy |
| 难度 | APPRENTICE |
| 模块 | SQL 注入 |
| 日期 | 2026-09-16 |
| 耗时 | 待补充 |

## 一、漏洞原理

商品列表的查询由服务端拼成：

```sql
SELECT * FROM products WHERE category = 'Accessories' AND released = 1
```

`released = 1` 是「只显示已上架商品」的过滤条件，而 `category` 是**用户可控**的输入。

因为 `category` 的值被**直接拼接进 SQL 字符串**，输入里的单引号会提前闭合这个字符串字面量。再补一个注释符 `--`，后面的 `AND released = 1` 整段就被注释掉了：

```sql
SELECT * FROM products WHERE category = 'Accessories'--' AND released = 1
                                                    ↑ 从这里之后全被注释
```

查询条件只剩 `category = 'Accessories'`，未上架商品被一并返回。

## 二、复现步骤

1. 点击 `Accessories` 分类，确认正常查询只返回已上架商品
2. 用 Burp Suite 抓这个请求，参数形如 `category=Accessories`
3. 把参数改为：

```
category=Accessories'--
```

4. 放行请求，观察响应：商品列表里**多出了未上架的商品**

**关键点**：`--` 把「已上架」这个过滤条件注掉了。注意 `--` 后需要一个空格才构成注释（URL 里通常写 `--+` 或 `--%20`），否则某些数据库不认。

## 三、根本原因

用户输入被当成 **SQL 代码**，而不是**数据**。

数据库收到的是完整字符串，它无法区分哪部分是开发者写的逻辑、哪部分是用户输入的内容。所以输入里的引号就获得了改变 SQL 语义的能力。

这不是「某个字符忘了过滤」，而是**字符串拼接这种构造方式本身**就给注入留了门。

## 四、修复方案

1. **参数化查询（预编译）** —— `PreparedStatement`／各语言的参数绑定。SQL 结构先编译固定，用户输入只作为「值」传入，永远无法改变语句结构。**这是根治手段**
2. **最小权限** —— 数据库账号不给无关表的读权限，缩小注入成功后的影响面
3. **不把黑名单过滤当主防线** —— 转义／过滤引号容易被编码、大小写、注释等方式绕过，只能算纵深防御的一层

## 五、面试问答

**Q：为什么一个单引号就能改变查询语义？**
A：因为输入被拼进了 SQL 字面量，单引号提前闭合了字符串，后面的内容被数据库当作 SQL 语法解析。

**Q：参数化查询和字符串拼接的本质区别？**
A：拼接是把用户输入混进 SQL 文本一起解析，输入的语法含义会被执行；参数化先固定语句结构，再把输入当纯数据传入，语法含义被彻底消除。

**Q：只过滤单引号够不够？**
A：不够。还有数字型注入（不需要引号）、宽字节、编码变形等绕过。过滤是「堵」，参数化才是「根治」。

## 六、参考

- [Lab: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)

## 标签

`#SQL注入` `#WHERE子句` `#注释符` `#入门`
