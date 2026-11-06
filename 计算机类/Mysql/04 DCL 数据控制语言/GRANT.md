# GRANT —— 授予权限

> [!abstract] 一句话
> `GRANT` 是 DCL 里的"**发钥匙**"语句：把"对某个库 / 某张表 / 某一列的某种操作"授权给一个账号或角色。

> [!info] 本篇在 DCL 中的位置
> **GRANT 发权限** → [[REVOKE]] 收权限 → [[ROLE]] 把一组权限打包发放。
> 四类 SQL 的分工见 [[四大SQL分类讲解]]。

---

## 一、语法骨架

```sql
-- ① 给账号授权限
GRANT 权限列表
    ON [对象类型] 权限范围
    TO '用户名'@'主机' [, ...]
    [WITH GRANT OPTION];

-- ② 把角色授予账号（角色 = 权限包）
GRANT '角色名' [, ...]
    TO '用户名'@'主机' [, ...]
    [WITH ADMIN OPTION];
```

| 部分 | 是否必须 | 说明 |
| --- | --- | --- |
| `权限列表` | ✅ | 可写多个，逗号分隔；也可写 `ALL [PRIVILEGES]` |
| `ON ...` | ✅（授权限时） | 指定作用范围；授角色时**没有** `ON` 子句 |
| `TO 账号` | ✅ | 账号必须写成 `'user'@'host'` |
| `WITH GRANT OPTION` | ❌ | 允许对方把权限继续转授给别人 |
| `WITH ADMIN OPTION` | ❌ | 角色专用，允许对方把角色继续授予别人 |

> [!tip] 一眼区分两种 `GRANT`
> **有 `ON` → 授权限；没有 `ON` → 授角色。**
> `GRANT SELECT ON db.* TO u` 是发权限；
> `GRANT 'r_read' TO u` 是发角色。

> [!warning] MySQL 8.0 不能再"顺手建用户"
> 5.7 及以前可以写 `GRANT ... TO u IDENTIFIED BY 'pwd'`，顺带创建账号。
> **8.0 起这个写法被移除**，必须先 `CREATE USER`，再 `GRANT`。
> 对不存在的账号直接授权，会报 `ERROR 1410: You are not allowed to create a user with GRANT`。

---

## 二、账号怎么写：`'user'@'host'`

MySQL 的账号是"**用户名 + 来源主机**"的组合，两部分都相同才算同一个账号。

```sql
CREATE USER 'app_user'@'192.168.1.%' IDENTIFIED BY 'StrongPass_2026!';
```

| 写法 | 含义 | 松紧 |
| --- | --- | --- |
| `'u'@'localhost'` | 本机（Unix 下通常走 socket） | 紧 |
| `'u'@'127.0.0.1'` | 本机 TCP | 紧 |
| `'u'@'192.168.1.%'` | 指定网段 | 中 |
| `'u'@'db01.example.com'` | 指定域名 | 紧 |
| `'u'@'%'` | 任意主机 | ==最松== |

> [!warning] `localhost` 和 `127.0.0.1` 不是一回事
> - `localhost` 在 Unix 上优先走 **Unix socket**，不走 TCP；
> - `127.0.0.1` 才走 TCP。
>
> 所以 `'u'@'localhost'` 能连、`'u'@'127.0.0.1'` 连不上（或反过来）是常见现象，排查时不要想当然。

> [!note] `%` 也不是"包打天下"
> 一个连接可能同时匹配多个账号，MySQL 会按 **host 的具体程度**排序，越具体的越优先。
> `''@'localhost'` 这类**匿名账号**也可能插队，导致"密码明明对却 Access denied"。
>
> 排查第一步永远是：
> ```sql
> SELECT user, host FROM mysql.user ORDER BY user, host;
> ```

> [!tip] 主机名解析
> 用域名做 host 时，服务器要做反向 DNS。生产环境常设 `skip_name_resolve=ON`，此时 host 只认 IP / `localhost`，域名写法会失效。

---

## 三、权限范围：五个层级

| 层级 | 写法 | 示例 | 影响面 |
| --- | --- | --- | --- |
| 全局 | `*.*` | `GRANT SELECT ON *.* TO u` | 所有库、所有表 |
| 单库 | `db.*` | `GRANT SELECT ON school.* TO u` | 整个库 |
| 单表 | `db.tbl` | `GRANT SELECT ON school.student TO u` | 一张表 |
| 单列 | `db.tbl(col, ...)` | `GRANT SELECT (name, score) ON school.student TO u` | 表的某几列 |
| 存储程序 | `db.routine` | `GRANT EXECUTE ON PROCEDURE school.p_stats TO u` | 一个过程 / 函数 |

```sql
GRANT SELECT ON *.* TO 'monitor'@'%';                 -- 全局只读
GRANT SELECT ON school.* TO 'report'@'%';             -- 单库只读
GRANT SELECT, UPDATE ON school.student TO 'app'@'%';  -- 单表读写
GRANT SELECT (name, score) ON school.student TO 'ta'@'%'; -- 单列只读
GRANT EXECUTE ON PROCEDURE school.p_stats TO 'report'@'%';
```

> [!important] 收回时必须回到**同一层级**
> 在 `school.*` 上授的权限，不能用 `REVOKE ... ON school.student` 收回。
> 授权和收权的"作用范围"必须一一对应，详见 [[REVOKE]]。

> [!note] 列级权限的限制
> 列级只支持 `SELECT`、`INSERT`、`UPDATE`、`REFERENCES` 这几种。
> `GRANT OPTION` **不能**授到列级；`ALL PRIVILEGES` 在列级也没有意义。

> [!warning] MySQL 没有原生"行级权限"
> 权限最细只到**列**。想实现"每个销售只能看自己的订单"，要靠：
> - 视图 + `WHERE` 过滤，再把视图权限给用户；
> - 或应用层拼接查询条件。
>
> 不要幻想 `GRANT ... WHERE user_id = CURRENT_USER()` 这种语法，MySQL 不支持。

---

## 四、常用权限清单

### 4.1 数据与结构

| 权限 | 作用 | 常见层级 |
| --- | --- | --- |
| `SELECT` | 查询 | 全局 / 库 / 表 / 列 |
| `INSERT` | 插入 | 全局 / 库 / 表 / 列 |
| `UPDATE` | 修改 | 全局 / 库 / 表 / 列 |
| `DELETE` | 删除行 | 全局 / 库 / 表 |
| `CREATE` | 建库、建表 | 全局 / 库 |
| `ALTER` | 改表结构 | 全局 / 库 / 表 |
| `DROP` | 删库、删表 | 全局 / 库 / 表 |
| `INDEX` | 建 / 删索引 | 全局 / 库 / 表 |
| `REFERENCES` | 建外键时引用 | 全局 / 库 / 表 / 列 |
| `CREATE TEMPORARY TABLES` | 建临时表 | 全局 / 库 |
| `LOCK TABLES` | 显式锁表 | 全局 / 库 |

### 4.2 视图、程序、管理

| 权限 | 作用 | 常见层级 |
| --- | --- | --- |
| `CREATE VIEW` | 建视图 | 全局 / 库 / 表 |
| `SHOW VIEW` | 查看视图定义 | 全局 / 库 / 表 |
| `CREATE ROUTINE` | 建存储过程 / 函数 | 全局 / 库 |
| `ALTER ROUTINE` | 改 / 删存储程序 | 全局 / 库 |
| `EXECUTE` | 调用存储过程 / 函数 | 全局 / 库 / 程序 |
| `TRIGGER` | 建 / 删触发器 | 全局 / 库 / 表 |
| `EVENT` | 建 / 改 / 删事件 | 全局 / 库 |
| `CREATE USER` | 建 / 删 / 改名用户，`REVOKE ALL` | 全局 |
| `PROCESS` | `SHOW PROCESSLIST`、看全部会话 | 全局 |
| `RELOAD` | `FLUSH` 类操作 | 全局 |
| `SHUTDOWN` | 关闭实例 | 全局 |
| `FILE` | 读写服务器文件、`LOAD DATA` | 全局 |
| `REPLICATION CLIENT` | 查主从状态 | 全局 |
| `REPLICATION SLAVE` | 作为从库同步 | 全局 |
| `SHOW DATABASES` | 看到所有库名 | 全局 |
| `GRANT OPTION` | 转授权 | 全局 / 库 / 表 |

> [!info] 动态权限（8.0）
> 8.0 把 `SUPER` 拆成了一组**动态权限**，例如：
> `SYSTEM_VARIABLES_ADMIN`、`SESSION_VARIABLES_ADMIN`、`BACKUP_ADMIN`、`BINLOG_ADMIN`、`CONNECTION_ADMIN`、`SYSTEM_USER`、`AUDIT_ADMIN`。
>
> 动态权限**只能全局授予**，写法是 `GRANT BACKUP_ADMIN ON *.* TO u;`。
> 它们通常只给运维 / DBA 账号，业务账号一个都不要给。

### 4.3 三个容易误解的写法

| 写法 | 真实含义 |
| --- | --- |
| `ALL [PRIVILEGES]` | 该层级上的**所有静态权限**；不含动态权限，也不含 `GRANT OPTION` |
| `USAGE` | "**没有任何权限**"，只表示账号存在、能登录；历史上常用来占位建号 |
| `SUPER` | 8.0 中已弃用，被拆成多个动态权限；新项目不要再用 |

> [!tip] `GRANT ALL ON *.*` 到底给了什么？
> 给了**所有静态全局权限**：包括 `DROP`、`CREATE USER`、`FILE`、`PROCESS`、`SHUTDOWN`……
> 相当于把整个实例交给对方。==这是生产环境最危险的一条授权。==

---

## 五、常用授权示例

### 5.1 业务账号：最小授权

```sql
-- 1. 先建账号（8.0 不会自动创建）
CREATE USER 'app_user'@'192.168.1.%' IDENTIFIED BY 'StrongPass_2026!';

-- 2. 只给业务真正用到的权限：查、增、改
GRANT SELECT, INSERT, UPDATE ON school.* TO 'app_user'@'192.168.1.%';

-- 3. 验证
SHOW GRANTS FOR 'app_user'@'192.168.1.%';
```

> [!tip] 为什么不给 `DELETE`？
> 业务上"删除"通常应做成**逻辑删除**（`is_deleted` 标记）。
> 物理删除权限一旦给出去，误操作就是不可逆的数据事故。

### 5.2 只读报表账号

```sql
CREATE USER 'report'@'%' IDENTIFIED BY 'Report_2026!';
GRANT SELECT ON school.* TO 'report'@'%';
```

如果只想开放某几张表：

```sql
GRANT SELECT ON school.student TO 'report'@'%';
GRANT SELECT ON school.score   TO 'report'@'%';
```

### 5.3 单列授权

```sql
-- 只能看姓名和分数，看不到身份证、手机号
GRANT SELECT (name, score) ON school.student TO 'ta'@'%';
```

> [!note] 列级权限适合"脱敏"场景
> 表里有敏感字段（身份证、薪资）时，与其把整表 `SELECT` 给出去再靠应用层遮掩，不如只授必要的列。
> 但列级权限会让 `SELECT *` 直接失败，接入前要和业务方确认。

### 5.4 存储过程执行权

```sql
GRANT EXECUTE ON PROCEDURE school.p_stats TO 'report'@'%';
GRANT EXECUTE ON FUNCTION  school.f_score_level TO 'app_user'@'%';
```

> [!tip] "只给 `EXECUTE`" 是一种收口手段
> 业务不直接碰表，只调用存储过程，可以把增删改的逻辑封装在过程里。
> 但要注意：过程的 `SQL SECURITY` 决定它按谁的权限执行，别把 `DEFINER` 设成 `root` 后就以为安全了。

### 5.5 监控 / 备份类账号

```sql
CREATE USER 'monitor'@'10.0.0.%' IDENTIFIED BY 'Monitor_2026!';
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'monitor'@'10.0.0.%';

CREATE USER 'backup'@'localhost' IDENTIFIED BY 'Backup_2026!';
GRANT BACKUP_ADMIN ON *.* TO 'backup'@'localhost';
```

### 5.6 给角色配权限

```sql
CREATE ROLE 'r_read';
GRANT SELECT ON school.* TO 'r_read';

GRANT 'r_read' TO 'report'@'%';
SET DEFAULT ROLE 'r_read' TO 'report'@'%';
```

角色本身怎么用，见 [[ROLE]]。

---

## 六、`WITH GRANT OPTION`：转授权

```sql
GRANT SELECT ON school.* TO 'leader'@'%' WITH GRANT OPTION;
```

之后 `leader` 可以把 `SELECT ON school.*` 再授给别人：

```sql
-- 以 leader 身份执行
GRANT SELECT ON school.* TO 'report'@'%';
```

`SHOW GRANTS` 里会多出这句：

```sql
GRANT SELECT ON `school`.* TO `leader`@`%` WITH GRANT OPTION
```

> [!danger] 转授权是权限扩散的源头
> 一旦有了 `GRANT OPTION`，权限就能**沿着链条无限传下去**：
> `DBA → leader → 组长 → 实习生`。
> 中途收回 `leader` 的权限，**不会自动收回**他授出去的那些（见 [[REVOKE]]）。
> 除非确有"分级管理"的需求，否则不要加。

> [!info] 8.0.16 起语义收窄
> 早年 `WITH GRANT OPTION` 会把授权者在该层级的**全部权限**的转授权一起给出，容易放大权限。
> 8.0.16 起改为：**只对本次语句列出的权限**给转授权。
> 跨版本迁移时，别假设行为一致。

> [!tip] 只收回"转授权"，保留权限本身
> ```sql
> REVOKE GRANT OPTION ON school.* FROM 'leader'@'%';
> ```
> 执行后 `leader` 仍然有 `SELECT`，只是不能再转授。

角色版本的对应写法是 `WITH ADMIN OPTION`：

```sql
GRANT 'r_read' TO 'leader'@'%' WITH ADMIN OPTION;
-- 收回
REVOKE ADMIN OPTION FOR 'r_read' FROM 'leader'@'%';
```

---

## 七、授权之后怎么验证

```sql
-- 看某个账号的权限
SHOW GRANTS FOR 'app_user'@'192.168.1.%';

-- 看当前登录账号
SHOW GRANTS FOR CURRENT_USER();

-- 连带角色一起看（8.0）
SHOW GRANTS FOR 'report'@'%' USING 'r_read';
```

输出解读：

| 输出 | 含义 |
| --- | --- |
| `GRANT USAGE ON *.* TO ...` | 账号存在，但**没有任何权限** |
| ``GRANT SELECT ON `school`.* TO ...`` | 库级只读 |
| `... WITH GRANT OPTION` | 该权限还能转授 |
| ``GRANT `r_read`@`%` TO ...`` | 持有角色（不是直接权限） |

> [!warning] 不要直接 `SELECT * FROM mysql.user`
> 8.0 的 `mysql.user` 把权限拆到了多张表：`user`、`db`、`tables_priv`、`columns_priv`、`procs_priv`、`role_edges` 等。
> 直接查容易漏、也容易误读。用 `SHOW GRANTS`，或者查 `information_schema` 的 `USER_PRIVILEGES` / `SCHEMA_PRIVILEGES` / `TABLE_PRIVILEGES`。

---

## 八、权限什么时候生效

> [!success] 正常情况：立刻生效
> `GRANT` / `REVOKE` 会同步更新内存中的权限结构，**下一条语句就能看到变化**。
> 不需要重启，不需要重连，也**不需要 `FLUSH PRIVILEGES`**。

| 场景 | 是否要额外操作 |
| --- | --- |
| 用 `GRANT` / `REVOKE` 改权限 | ❌ 自动生效 |
| 直接 `INSERT/UPDATE/DELETE` 改 `mysql` 库权限表 | ✅ 必须 `FLUSH PRIVILEGES` |
| 改了角色权限 | ❌ 对持有该角色的会话立即生效（角色需已激活） |
| 修改了账号的默认角色 | 新会话生效；当前会话可用 `SET ROLE` 调整 |

> [!warning] `FLUSH PRIVILEGES` 不是仪式
> 它做的是"**重新加载权限表**"。
> 正常授权流程根本用不到它；反复执行只会让人误以为"授权必须刷新"。
> 只有当你绕过 `GRANT`，直接改 `mysql.*` 系统表时，它才是必需品。

---

## 九、`GRANT` 会隐式提交

`GRANT`、`REVOKE`、`CREATE USER`、`DROP USER` 等**账号管理语句**都会隐式提交当前事务。

```sql
START TRANSACTION;

INSERT INTO student (name, score) VALUES ('临时数据', 60);
-- 本想着可以 ROLLBACK

GRANT SELECT ON school.* TO 'report'@'%';
-- ⚠️ 隐式提交：上面的 INSERT 被一起提交了

ROLLBACK;
-- 撤不回那条 INSERT
```

> [!important] 账号管理语句清单（都会隐式提交）
> `CREATE USER` / `ALTER USER` / `DROP USER` / `RENAME USER`
> `GRANT` / `REVOKE`
> `CREATE ROLE` / `DROP ROLE` / `SET DEFAULT ROLE`
> `SET PASSWORD`
>
> ==不要把这些语句和业务 DML 写进同一个事务。==

---

## 十、安全红线

> [!danger] 六条不要
> 1. **不要用 `root` 跑业务**。应用账号只给业务所需的最小权限。
> 2. **不要 `GRANT ALL ON *.*`**。这等于把实例交给对方，包括 `DROP` 和 `CREATE USER`。
> 3. **不要随手 `WITH GRANT OPTION`**。转授权会让权限失控扩散。
> 4. **不要 `'user'@'%'`**，能限定 IP / 网段就限定。
> 5. **不要给业务账号 DDL 权限**（`CREATE` / `ALTER` / `DROP`）。
> 6. **不要把动态权限当普通权限发**（`SYSTEM_VARIABLES_ADMIN`、`BACKUP_ADMIN` 等）。

> [!tip] 一套可落地的账号分层
> | 账号 | 权限 |
> | --- | --- |
> | 业务应用 | `SELECT` / `INSERT` / `UPDATE` / `DELETE`（单库） |
> | 报表只读 | `SELECT`（单库或单表） |
> | 迁移 / DBA | DDL 权限，独立账号，禁止业务使用 |
> | 监控 | `PROCESS` / `REPLICATION CLIENT` |
> | 备份 | `BACKUP_ADMIN` / `RELOAD` |
>
> 需要同一套权限发给多个账号时，**用角色**，不要逐个复制 `GRANT`。

> [!note] 认证插件也影响"能不能连上"
> MySQL 8.0 默认 `caching_sha2_password`。老客户端 / 老驱动可能不认，报 `Authentication plugin 'caching_sha2_password' cannot be loaded`。
> 解决方向是升级驱动，而不是无脑改回 `mysql_native_password`；确需兼容时再显式指定：
> ```sql
> CREATE USER 'legacy'@'%'
>   IDENTIFIED WITH mysql_native_password BY 'StrongPass_2026!';
> ```

---

## 十一、常见报错与对策

> [!failure] ERROR 1410：You are not allowed to create a user with GRANT
> 8.0 中给不存在的账号授权。先 `CREATE USER`，再 `GRANT`。

> [!failure] ERROR 1133：Can't find any matching row in the user table
> 同样是账号不存在（多见于 5.7 关闭自动建号后的写法）。检查 `'user'@'host'` 是否完全一致。

> [!failure] ERROR 1044：Access denied for user ... to database ...
> 执行 `GRANT` 的账号自己**没有 `GRANT OPTION`**，或没有对应的权限。
> 授权者只能授出"自己拥有且能转授"的权限。

> [!failure] ERROR 1221：Incorrect usage of DB GRANT and GLOBAL PRIVILEGES
> 把**只能全局授予**的权限写到了库级，例如 `GRANT FILE ON school.* TO u;`。
> 改成 `ON *.*`，或换掉该权限。

> [!failure] ERROR 1064：语法错误
> 8.0 中最常见的原因是还在用 `GRANT ... IDENTIFIED BY ...`。
> 拆成 `CREATE USER` + `GRANT` 两步。

> [!failure] ERROR 1045：Access denied for user (using password: YES)
> 连不上，不是授权问题。检查账号是否存在、host 是否匹配、密码是否正确、认证插件是否兼容。

> [!failure] ERROR 1698：Access denied for user 'root'@'localhost'
> 常见于 Debian / Ubuntu 的 `auth_socket` 插件：`root` 只能用系统用户身份登录，不能用密码。
> 改用 `sudo mysql`，或调整认证插件。

> [!failure] ERROR 1820：You must reset your password using ALTER USER
> 密码已过期。执行 `ALTER USER 'u'@'h' IDENTIFIED BY '新密码';`。

---

## 十二、常见面试问答

> [!question] Q1：`GRANT` 的基本语法是什么？
> `GRANT 权限 ON 范围 TO 'user'@'host' [WITH GRANT OPTION];`
> 权限可以是单个、多个或 `ALL PRIVILEGES`；范围可以是 `*.*`、`db.*`、`db.tbl`、`db.tbl(col)`。

> [!question] Q2：MySQL 的权限分几个层级？
> 全局（`*.*`）、库（`db.*`）、表（`db.tbl`）、列（`db.tbl(col)`）、存储程序（`db.routine`）。
> 权限检查时按"从具体到全局"匹配。

> [!question] Q3：`WITH GRANT OPTION` 是什么？
> 允许被授权者把**同一权限**继续转授给他人。它是权限扩散的主要来源，非必要不加。
> 角色的对应概念是 `WITH ADMIN OPTION`。

> [!question] Q4：`GRANT` 之后需要 `FLUSH PRIVILEGES` 吗？
> 不需要。`GRANT` / `REVOKE` 会自动更新内存权限。
> 只有直接改 `mysql` 库权限表后，才需要手动刷新。

> [!question] Q5：为什么给用户授了权限，他还是连不上？
> 授权解决"能做什么"，连接还取决于"能不能登录"：
> 账号是否存在、`user@host` 是否匹配、密码 / 认证插件、账号是否被锁、密码是否过期。
> 先看 `ERROR` 编号，再分别排查。

> [!question] Q6：`ALL PRIVILEGES` 包含 `GRANT OPTION` 吗？
> 不包含。`GRANT ALL ON *.* TO u` 不含转授权，也不含动态权限。
> 要转授权必须显式写 `WITH GRANT OPTION`。

> [!question] Q7：能不能只给一张表的某几列授权？
> 可以，列级支持 `SELECT`、`INSERT`、`UPDATE`、`REFERENCES`：
> `GRANT SELECT (name, score) ON school.student TO u;`
> 但用户执行 `SELECT *` 会失败。

> [!question] Q8：`GRANT` 和事务有关系吗？
> 有。`GRANT` / `REVOKE` 等账号管理语句会**隐式提交**当前事务。
> 所以不要把授权和业务 DML 混在一个事务里。

> [!question] Q9：怎么查看一个用户最终有哪些权限？
> ```sql
> SHOW GRANTS FOR 'u'@'h';
> SHOW GRANTS FOR 'u'@'h' USING 'r_read';   -- 连带角色
> ```
> 输出里的 `USAGE` 表示"无任何权限"。

> [!question] Q10：为什么推荐用角色而不是逐个账号授权？
> 权限定义只维护一份，增删用户只是"发角色 / 收角色"，变更影响面清晰。
> 详见 [[ROLE]]。

---

> [!quote] 一句话记忆
> **`GRANT` 发钥匙：权限 + 范围 + 账号，三样缺一不可；**
> **`WITH GRANT OPTION` 是"配钥匙的许可"，能给就别给。**

---

## 相关笔记

- [[REVOKE]] —— 收回权限；授权与收权必须同层级
- [[ROLE]] —— 把一组权限打包，批量发放
- [[四大SQL分类讲解]] —— DCL 在四类 SQL 中的位置
- [[Create Table]] —— 被授权的对象从哪来
- [[Select]] · [[Insert]] · [[Update]] · [[Delete]] —— 最常被授予的四类数据权限
- [[存储过程]] · [[存储函数]] —— `EXECUTE` 权限的授予对象
- [[BEGIN]] · [[COMMIT]] · [[ROLLBACK]] —— 理解 DCL 的隐式提交
