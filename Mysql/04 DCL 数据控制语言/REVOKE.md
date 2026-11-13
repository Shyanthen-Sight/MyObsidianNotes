# REVOKE —— 收回权限

> [!abstract] 一句话
> `REVOKE` 是 DCL 里的"**收钥匙**"语句：把已经授予账号或角色的权限收回来。
> 和 [[GRANT]] 相反，但**授权在哪一层，收权就必须在哪一层**。

> [!info] 本篇在 DCL 中的位置
> [[GRANT]] 发权限 → **REVOKE 收权限** → [[ROLE]] 批量管理权限包。
> 四类 SQL 的分工见 [[四大SQL分类讲解]]。

---

## 一、语法骨架

```sql
-- ① 收回具体权限（必须和授权时的层级一致）
REVOKE 权限列表
    ON [对象类型] 权限范围
    FROM '用户名'@'主机' [, ...]
    [IGNORE UNKNOWN USER];

-- ② 收回全部权限和转授权（不写 ON）
REVOKE ALL [PRIVILEGES], GRANT OPTION
    FROM '用户名'@'主机' [, ...];

-- ③ 只收回转授权，保留权限本身
REVOKE GRANT OPTION
    ON [对象类型] 权限范围
    FROM '用户名'@'主机' [, ...];

-- ④ 收回角色
REVOKE '角色名' [, ...]
    FROM '用户名'@'主机' [, ...];

-- ⑤ 收回角色的转授权（ADMIN OPTION）
REVOKE ADMIN OPTION FOR '角色名'
    FROM '用户名'@'主机' [, ...];
```

| 写法 | 收什么 | 是否保留账号 |
| --- | --- | --- |
| `REVOKE 权限 ON 范围` | 指定层级上的指定权限 | ✅ |
| `REVOKE ALL PRIVILEGES, GRANT OPTION FROM` | **所有层级**的静态权限 + 转授权 | ✅ |
| `REVOKE GRANT OPTION ON 范围` | 只收"转授权"，权限还在 | ✅ |
| `REVOKE '角色' FROM` | 账号与角色的绑定关系 | ✅ |
| `REVOKE ADMIN OPTION FOR '角色'` | 只收角色的转授权 | ✅ |

> [!tip] `REVOKE` 从来不删除账号
> 它只改"权限"，账号还活着，还能登录（只是登录后可能什么都干不了）。
> 真要删人，用 `DROP USER`；要临时停用，用 `ALTER USER ... ACCOUNT LOCK`。

---

## 二、核心规则：授权在哪一层，收权就在哪一层

这是 `REVOKE` 最容易踩的坑。

| 当初怎么授权的 | 必须怎么收回 | 错误写法 |
| --- | --- | --- |
| `GRANT SELECT ON *.*` | `REVOKE SELECT ON *.*` | `REVOKE SELECT ON school.*` ❌ |
| `GRANT SELECT ON school.*` | `REVOKE SELECT ON school.*` | `REVOKE SELECT ON school.student` ❌ |
| `GRANT SELECT ON school.student` | `REVOKE SELECT ON school.student` | `REVOKE SELECT ON school.*` ❌ |
| `GRANT SELECT (name) ON school.student` | `REVOKE SELECT (name) ON school.student` | 不带列名 ❌ |
| `GRANT EXECUTE ON PROCEDURE school.p` | `REVOKE EXECUTE ON PROCEDURE school.p` | 写成表名 ❌ |

```sql
-- ✅ 对应关系正确
GRANT  SELECT ON school.* TO 'app'@'%';
REVOKE SELECT ON school.* FROM 'app'@'%';

-- ❌ 层级不一致：库级授的，表级收不回来
REVOKE SELECT ON school.student FROM 'app'@'%';
-- ERROR 1147: There is no such grant defined for user 'app' on host '%' on table 'student'
```

> [!important] 为什么必须同层级
> MySQL 的权限是**按层级分行存储**的：全局存在 `mysql.user`，库级存在 `mysql.db`，表级存在 `mysql.tables_priv`，列级存在 `mysql.columns_priv`。
> 收回时如果层级对不上，就找不到那条记录，自然报"没有这个授权"。
> 正确的做法是：**先用 `SHOW GRANTS` 看清授权层级，再原样收回。**

> [!note] 一条 `ALL` 授权可以拆开收
> `GRANT ALL ON school.* TO u` 在内部会被展开成一条条具体权限。
> 所以可以单独收回其中一项：
> ```sql
> REVOKE DELETE ON school.* FROM u;   -- 只收 DELETE，其余仍在
> ```
> `SHOW GRANTS` 会把剩下的权限重新列出来。

> [!warning] 收回比授权更"苛刻"
> `GRANT` 允许你在更粗的层级授予（例如给 `school.*`），
> 但 `REVOKE` **不允许**在更细的层级收（例如 `school.student`）。
> 唯一的例外是 `partial_revokes`，见本文第六节。

---

## 三、常用示例

### 3.1 收回单表 / 单库权限

```sql
-- 收回某张表的删除权限
REVOKE DELETE ON school.student FROM 'app_user'@'192.168.1.%';

-- 收回整个库的写权限
REVOKE INSERT, UPDATE, DELETE ON school.* FROM 'app_user'@'192.168.1.%';
```

### 3.2 收回列级权限

```sql
-- 当初：GRANT SELECT (name, score) ON school.student TO 'ta'@'%';
REVOKE SELECT (name, score) ON school.student FROM 'ta'@'%';
```

> [!note] 列级权限要带列名
> 不带列名的 `REVOKE SELECT ON school.student` 收的是**表级** `SELECT`，和列级授权对不上。
> 如果同时存在表级和列级授权，两者要分别收回。

### 3.3 收回全部权限

```sql
-- 收回所有层级的静态权限 + 转授权（账号保留）
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'app_user'@'192.168.1.%';
```

验证：

```sql
SHOW GRANTS FOR 'app_user'@'192.168.1.%';
-- GRANT USAGE ON *.* TO `app_user`@`192.168.1.%`
-- USAGE = 账号还在，但没有任何权限
```

### 3.4 只收回转授权

```sql
-- 保留 SELECT，只取消"再授权给别人"的能力
REVOKE GRANT OPTION ON school.* FROM 'leader'@'%';
```

### 3.5 收回角色

```sql
REVOKE 'r_read' FROM 'report'@'%';
```

> [!note] 收回角色 ≠ 删除角色
> 角色本身和它的权限定义都还在，只是这个账号不再持有它。
> 其他还持有该角色的账号不受影响。
> 要让角色彻底消失，用 `DROP ROLE 'r_read';`，见 [[ROLE]]。

### 3.6 收回后一定要验证

```sql
SHOW GRANTS FOR 'app_user'@'192.168.1.%';
-- 如果有角色，再连带角色看一次：
SHOW GRANTS FOR 'app_user'@'192.168.1.%' USING 'r_read';
```

> [!tip] "收回成功"不等于"权限没了"
> 如果账号通过**角色**间接持有权限，收回直接权限后它仍然有权限。
> 验证时要同时看直接授权和角色授权。

---

## 四、`REVOKE ALL` 到底收回了什么

MySQL 里有两种"收回全部"，效果不一样：

| 语句 | 收回范围 |
| --- | --- |
| `REVOKE ALL ON *.* FROM u` | 只收回**全局层级**的权限 |
| `REVOKE ALL PRIVILEGES, GRANT OPTION FROM u` | 收回**所有层级**的静态权限 + 转授权 |

> [!important] 别把两者混用
> 如果账号在 `school.*` 上还有库级权限，`REVOKE ALL ON *.*` **收不到**它。
> 要清空一个账号的全部权限，用不带 `ON` 的那条。

`REVOKE ALL PRIVILEGES, GRANT OPTION` 的边界：

| 对象 | 是否收回 |
| --- | --- |
| 全局 / 库 / 表 / 列 / 程序级静态权限 | ✅ |
| `GRANT OPTION` | ✅ |
| **动态权限**（`BACKUP_ADMIN` 等） | ❌ 需要按名收回 |
| **角色** | ❌ 需要 `REVOKE '角色' FROM` |
| 账号本身 | ❌ 账号保留 |

```sql
-- 动态权限单独收
REVOKE BACKUP_ADMIN ON *.* FROM 'backup'@'localhost';

-- 角色单独收
REVOKE 'r_read' FROM 'report'@'%';
```

> [!warning] 权限要求
> 执行 `REVOKE ALL PRIVILEGES, GRANT OPTION` 需要 **`CREATE USER` 权限**，或对 `mysql` 库有 **`UPDATE` 权限**。
> 普通业务账号没有这个能力。

> [!note] 收完权限，账号还能登录
> 如果目标是"这个人不能再用这个账号"，只 `REVOKE` 是不够的。
> 还应该：
> ```sql
> ALTER USER 'u'@'h' ACCOUNT LOCK;      -- 临时锁定
> ALTER USER 'u'@'h' IDENTIFIED BY '新随机密码';  -- 强制改密
> -- 或彻底删除
> DROP USER 'u'@'h';
> ```

---

## 五、转授权链：`REVOKE` 不会级联

假设有一条授权链：

```text
DBA ──GRANT SELECT WITH GRANT OPTION──▶ leader
leader ──GRANT SELECT──▶ report
```

现在 DBA 收回 `leader` 的权限：

```sql
REVOKE SELECT ON school.* FROM 'leader'@'%';
```

结果：

| 账号 | 收回后 |
| --- | --- |
| `leader` | ❌ 失去 `SELECT` |
| `report` | ✅ **仍然保留** `SELECT` |

> [!danger] MySQL 的 `REVOKE` 不会自动清理下游
> 收回某个账号的权限，**不会**收回它转授给别人的权限。
> 这在权限审计里叫"**孤儿授权**"：授权链的中间节点没了，末端还在。
>
> 所以清理权限时，不能只盯着一个人，要顺藤摸瓜：
> ```sql
> -- 逐个账号检查，找出不该存在的授权
> SELECT user, host FROM mysql.user;
> SHOW GRANTS FOR 'report'@'%';
> ```
> 更稳妥的做法是**一开始就不要随意使用 `GRANT OPTION` / `ADMIN OPTION`**。

> [!tip] 预防胜于补救
> 权限分级管理时，明确"谁可以转授、转授范围是什么"。
> 一旦发现权限扩散，先 `SHOW GRANTS` 全量盘点，再逐个收回，最后再收回上游的 `GRANT OPTION`。

---

## 六、`partial_revokes`：层级规则的唯一例外

MySQL 8.0.16 引入了 **部分撤销（partial revokes）**：
允许在**全局授权**的基础上，单独撤销某个库的权限。

```sql
-- 1. 开启部分撤销能力（全局变量，需 SYSTEM_VARIABLES_ADMIN）
SET PERSIST partial_revokes = ON;

-- 2. 先给全局只读
GRANT SELECT ON *.* TO 'analyst'@'%';

-- 3. 再把敏感库收回来（层级不一致，但合法）
REVOKE SELECT ON secret_db.* FROM 'analyst'@'%';
```

`SHOW GRANTS` 会显示成两行：

```sql
GRANT SELECT ON *.* TO `analyst`@`%`
REVOKE SELECT ON `secret_db`.* FROM `analyst`@`%`
```

> [!important] 它解决什么问题
> 正常规则下，"给全局权限"和"排除某个库"无法同时表达。
> 以前只能逐个库授权，库一多就难维护；`partial_revokes` 让"全局授权 + 黑名单"成为可能。

> [!warning] 使用注意
> - 这是**全局开关**，影响整个实例的授权语义，开启前要评估现有权限。
> - 只有 **8.0.16 及以上**支持。
> - 关闭 `partial_revokes` 前，必须先清理所有部分撤销记录，否则可能失败。
> - 开了以后，`SHOW GRANTS` 会出现 `REVOKE ...` 行，审计脚本要能识别。

---

## 七、收回之后什么时候生效

> [!success] 正常情况：立即生效
> `REVOKE` 会同步更新内存权限结构，**下一条语句就按新权限校验**。
> 不需要重启，不需要重连，也**不需要 `FLUSH PRIVILEGES`**。

| 场景 | 说明 |
| --- | --- |
| 收回直接权限 | 立即生效 |
| 收回角色 | 新连接立即按新配置；已有连接建议重连或 `SET ROLE` 确认 |
| 修改角色自身的权限 | 对持有该角色的会话生效（角色需处于激活状态） |
| 直接改 `mysql` 库权限表 | 必须 `FLUSH PRIVILEGES` |

> [!warning] 已有连接的状态可能"看起来没变"
> 账号权限被收回后，已建立的连接不会自动断开。
> 如果业务侧仍能操作，先确认是不是：
> 1. 通过**角色**间接持有权限；
> 2. 连接用的是**另一个账号**（host 不同也算不同账号）；
> 3. 应用用了**连接池**，旧连接还没轮换。
>
> 必要时主动 `KILL` 会话，或让应用重启连接池。

---

## 八、`REVOKE` vs `DROP USER` vs `ACCOUNT LOCK`

| 操作 | 权限 | 账号 | 能否登录 | 适用场景 |
| --- | --- | --- | --- | --- |
| `REVOKE` | 收回指定权限 | 保留 | ✅ 能（可能无权限可用） | 调岗、降权、最小权限回归 |
| `REVOKE ALL PRIVILEGES, GRANT OPTION` | 清空静态权限 | 保留 | ✅ 能（`USAGE`） | 保留账号但完全停权 |
| `DROP USER` | 删除账号及其全部授权 | ❌ 删除 | ❌ | 离职、废弃账号 |
| `ALTER USER ... ACCOUNT LOCK` | 不改权限 | 保留 | ❌ 锁定 | 临时停用、疑似泄露 |
| `ALTER USER ... PASSWORD EXPIRE` | 不改权限 | 保留 | ⚠️ 需先改密 | 强制轮换密码 |

```sql
-- 临时停用
ALTER USER 'u'@'h' ACCOUNT LOCK;
-- 恢复
ALTER USER 'u'@'h' ACCOUNT UNLOCK;

-- 彻底删除（权限、角色绑定一并清理）
DROP USER 'u'@'h';
```

> [!danger] 不要用 `DELETE FROM mysql.user` 删账号
> 这样只删了账号行，`mysql.db`、`mysql.tables_priv`、`mysql.role_edges` 等表里可能留下**孤儿授权**。
> 如果之后又建了同名账号，可能"继承"这些残留权限。
> 永远用 `DROP USER`。

---

## 九、常见坑

> [!failure] 坑一：层级不一致，收了个寂寞
> 库级授的权限用表级收回，报 `ERROR 1147`；反过来也可能报 `ERROR 1141`。
> **先 `SHOW GRANTS`，再照抄层级。**

> [!failure] 坑二：以为收回权限 = 删除账号
> `REVOKE` 后账号仍可登录。要禁止登录，用 `ACCOUNT LOCK` 或 `DROP USER`。

> [!failure] 坑三：忘了角色
> `REVOKE ALL PRIVILEGES, GRANT OPTION` 不会收回角色。
> 账号通过角色仍然有权限。必须再执行 `REVOKE '角色' FROM 账号`。

> [!failure] 坑四：忘了动态权限
> `BACKUP_ADMIN`、`SYSTEM_VARIABLES_ADMIN` 等动态权限不在 `REVOKE ALL` 的范围内，要按名收回。

> [!failure] 坑五：以为会级联
> 收回上游账号的权限，不会影响它转授出去的下游账号。需要逐个清理。

> [!failure] 坑六：先收权限，后发现还要用
> 生产环境收回前先确认业务影响，最好先 `ACCOUNT LOCK` 观察，再决定是否彻底收回。

> [!failure] 坑七：在事务里执行 `REVOKE`
> `REVOKE` 会**隐式提交**当前事务，导致之前未提交的 DML 一起落盘。
> 不要把权限操作和业务事务混在一起。

> [!failure] 坑八：用错 host
> `'u'@'%'` 和 `'u'@'localhost'` 是两个账号。
> 给 `'u'@'%'` 收权，不影响 `'u'@'localhost'`。

> [!failure] 坑九：收回后没验证
> 收回脚本执行成功，不代表目标权限真的没了（可能有角色、有同名的另一个 host 账号）。
> 收完必须 `SHOW GRANTS` 复核。

---

## 十、安全与运维实践

### 10.1 离职 / 调岗的标准动作

```sql
-- 1. （可选）先锁定，观察是否有业务报错
ALTER USER 'alice'@'192.168.1.%' ACCOUNT LOCK;

-- 2. 收回直接权限
REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'alice'@'192.168.1.%';

-- 3. 收回角色
REVOKE 'r_app' FROM 'alice'@'192.168.1.%';

-- 4. 收回动态权限（如有）
REVOKE BACKUP_ADMIN ON *.* FROM 'alice'@'192.168.1.%';

-- 5. 确认
SHOW GRANTS FOR 'alice'@'192.168.1.%';

-- 6. 最终删除（确认无需保留时）
DROP USER 'alice'@'192.168.1.%';
```

> [!tip] 顺序建议
> **先锁、再收、后删**。
> 直接 `DROP USER` 会让排查失去线索；先锁定可以确认影响面，也方便回滚。

### 10.2 定期权限审计

```sql
-- 列出所有账号
SELECT user, host, account_locked, password_expired
  FROM mysql.user
 ORDER BY user, host;

-- 逐个看授权
SHOW GRANTS FOR 'app_user'@'%';
```

> [!success] 审计关注点
> 1. 是否存在 `'%'` 账号；
> 2. 是否有业务账号持有 DDL 权限；
> 3. 是否有 `WITH GRANT OPTION`；
> 4. 是否有长期未使用的账号；
> 5. 是否有 `GRANT ALL ON *.*`；
> 6. 角色绑定是否与实际岗位一致。

### 10.3 收回权限的原则

> [!important] 三条原则
> 1. **最小权限回归**：调岗后按新岗位重新授权，而不是在旧权限上做减法。
> 2. **能锁不删、能删不晾**：临时停用用锁，彻底不用就删，别留"僵尸账号"。
> 3. **变更可追溯**：谁在什么时候收了谁的什么权限，要有记录。

---

## 十一、常见报错与对策

> [!failure] ERROR 1141：There is no such grant defined for user ... on host ...
> 该账号在**这个层级**没有这条授权。检查 `SHOW GRANTS`，确认授权层级和权限名。

> [!failure] ERROR 1147：There is no such grant defined for user ... on host ... on table ...
> 表级收回时找不到对应授权。多半是当初在库级授予的，需要回到 `db.*` 层级收回。

> [!failure] ERROR 1144：Illegal GRANT/REVOKE command
> 语法组合不合法。常见于把 `ALL PRIVILEGES, GRANT OPTION` 和 `ON` 混写，或权限名写错。

> [!failure] ERROR 1221：Incorrect usage of DB GRANT and GLOBAL PRIVILEGES
> 把只能全局操作的权限写到了库级。检查权限的作用层级。

> [!failure] ERROR 1044：Access denied for user ... to database ...
> 执行 `REVOKE` 的账号自己权限不够。收回某权限，需要你**拥有该权限且拥有 `GRANT OPTION`**。

> [!failure] 对不存在的账号收回
> 会失败。MySQL 8.0.16+ 可以在语句末尾加 `IGNORE UNKNOWN USER` 跳过：
> ```sql
> REVOKE SELECT ON school.* FROM 'ghost'@'%' IGNORE UNKNOWN USER;
> ```

> [!failure] 角色相关报错
> 常见提示如 "Role ... is not granted to ..." 或 "Cannot revoke role ..."。
> 先用 `SHOW GRANTS FOR 'u'@'h'` 确认角色确实已授予，再检查角色名和 host 是否完全一致。

---

## 十二、常见面试问答

> [!question] Q1：`REVOKE` 的基本语法是什么？
> `REVOKE 权限 ON 范围 FROM 'user'@'host';`
> 清空全部用 `REVOKE ALL PRIVILEGES, GRANT OPTION FROM 'user'@'host';`
> 收角色用 `REVOKE 'role' FROM 'user'@'host';`

> [!question] Q2：为什么收回权限时层级必须一致？
> 因为权限按层级分别存放在不同系统表中（`user`、`db`、`tables_priv`、`columns_priv`）。
> 层级不一致就找不到对应记录，报 1141 / 1147。

> [!question] Q3：`REVOKE ALL ON *.*` 和 `REVOKE ALL PRIVILEGES, GRANT OPTION FROM` 有什么区别？
> 前者只收**全局层级**的权限；后者收**所有层级**的静态权限和转授权。
> 清空账号权限要用后者。

> [!question] Q4：收回权限会级联吗？
> 不会。MySQL 不会自动收回被授权者转授给第三方的权限。
> 需要逐个账号审计和清理。

> [!question] Q5：`REVOKE` 之后要 `FLUSH PRIVILEGES` 吗？
> 不需要。正常 `REVOKE` 会自动刷新内存权限。
> 只有直接改 `mysql` 系统表后才需要。

> [!question] Q6：`REVOKE` 和 `DROP USER` 有什么区别？
> `REVOKE` 只收回权限，账号还在、还能登录；
> `DROP USER` 删除账号及其全部授权，账号不存在。
> 临时停用则用 `ALTER USER ... ACCOUNT LOCK`。

> [!question] Q7：`REVOKE ALL PRIVILEGES` 会收回角色和动态权限吗？
> 都不会。角色要用 `REVOKE '角色' FROM` 收回；
> 动态权限（如 `BACKUP_ADMIN`）要按名称收回。

> [!question] Q8：`partial_revokes` 是干什么的？
> MySQL 8.0.16+ 的部分撤销能力：允许在全局授权的基础上，单独撤销某个库的权限，
> 实现"全局允许 + 敏感库排除"的授权模型。

> [!question] Q9：怎么确认权限真的被收回了？
> ```sql
> SHOW GRANTS FOR 'u'@'h';
> SHOW GRANTS FOR 'u'@'h' USING '角色名';
> ```
> 同时确认没有同名的其他 host 账号、没有通过角色间接持有权限。

> [!question] Q10：收回权限时如何避免影响业务？
> 先评估依赖、先用 `ACCOUNT LOCK` 或灰度账号验证；
> 变更窗口内操作，保留回滚脚本（重新 `GRANT` 的语句），收完立即复核。

---

> [!quote] 一句话记忆
> **`REVOKE` 收钥匙：层级要对、名字要对、host 要对；**
> **它只收权限不删人，不级联、不碰角色和动态权限。**

---

## 相关笔记

- [[GRANT]] —— 授予权限；授权与收权必须同层级
- [[ROLE]] —— 角色的创建、授予、激活与删除
- [[四大SQL分类讲解]] —— DCL 在四类 SQL 中的位置
- [[Create Table]] · [[Select]] · [[Insert]] · [[Update]] · [[Delete]] —— 常被授予 / 收回的权限对象
- [[BEGIN]] · [[COMMIT]] · [[ROLLBACK]] —— 理解 DCL 的隐式提交
