### 一. 基础点

#### 1. ACID vs BASE

**ACID** 是**事务四大特性**, 是MySQL, Oracle, PostgreSQL等*关系型数据库*的核心标准

1. A - Atomic 原子性: 事务是不可分割的最小单元, 要么全成功, 要么全失败回滚
2. C - Consistency 一致性: ACID的最终目标
3. I - Isolation 隔离性: 多个并发事务之间相互隔离, 互不干扰, 避免长度, 不可重复读, 幻读
4. D - Durability 持久性: 事务提交成功后, 修改永久落地磁盘

ACID 适合于 金融, 支付, 订单, 库存扣等要求**强一致**的业务, 容忍性能牺牲

**BASE** 是 Redis 相关

#### 2. 数据库三大范式

范式是**表结构设计规范**, 目的是减少数据冗余, 避免增删改异常; 范式越高冗余越少, 但是多表关联会变多

- 主属性: 候选键 (能当主键的字段集合) 里的字段;
- 非主属性: 不在候选键中的字段
- 函数依赖: 如果给定 X 的值, 就能唯一确定 Y 的值, 就成 Y 函数依赖于 X

**三大范式:**

1. 1NF : 数据表中**每一列的值必须是不可拆分的原子值**，不能是集合、数组、多个数据拼接
2. 2NF: 所有**非主属性必须完全依赖整个主键**，不能只依赖主键的一部分（消除部分依赖）。
   1. 每一列都和整个主键相关, 而不能至于主键的某一部分相关 (即不能"主键的其他部分不能决定该列值")
3. 3NF: **非主属性不能依赖其他非主属性**，只能直接依赖主键，消除传递依赖。
   1. 每一列数据都和主键直接相关, 而不能间接相关

#### 3. 外键约束

外键约束是为了维护表与表之间的关系, **确保数据的完整性和一致性, **若没有外键约束, 一个表的情况更新可能不会同步到另一个表.

#### 4. 常见的函数

##### 字符串函数

- `CONCAT(str1, str2, ...)`
- `LENGTH(str)`: 返回字符数
- `SUBSTRING(str, pos, len)`: 从指定位置开始, 截取指定长度的子串
- `REPLACE(str, from_str, to_str`: 将字符串中的某部分替换为另一个字符串

##### 数值函数

- `ABS(num)`
- `POWER(num, exponent)`: 返回指定数字的指定幂次方

##### 日期和时间函数

- `NOW()`: 当前日期 + 时间
- `CURDATE()`: 仅日期

##### 聚合函数

- `COUNT(column)`: 指定列非NULL值的个数
- `SUM(column/条件表达式)`
- `AVG(column)`
- `MAX(column)/MIN(column)`

- **`EXIST(子查询)`: ** 返回布尔值
  - 子查询能查出至少 1 条记录 -> 返回`true`
  - 一条都查不到 -> 返回`false`
  - 子查询经常用 `SELECT 1`, 是性能最优，行业规范
- `IN(子查询)`: 返回布尔值, `IN`前面的字段是否属于后面子查询的某行数据

#### 5. 外键约束

外键约束的作用是维护表与表之间的关系, **确保数据的完整性和一致性. ** 比如: 如果没有约束外键, 可能出现改表的一行数据对象已经删除了, 但是外键关联的那张表还是有指向该对象的外键值.

#### 6. SQL 查询语句的执行顺序

![Image_2026-06-25_18-56-59_zxrnm1yw.gv0](./imgs/Image_2026-06-25_18-56-59_zxrnm1yw.gv0-1782402053117-2.png)

### 二. 题目

#### MySQL如何避免重复插入数据

1. 使用 UNIQUE 对字段约束

2. **插入和更新结合**使用 `INSERT ... ON DUPLICATE KEY UPDATE`

   ```SQL
   INSERT INTO users (email, name)
   VALUES ('example@example.com', 'John Doe')
   ON DUPLICATE KEY UPDATE name = VALUES(name);
   ```

   email和name中要有一个用主键/UNIQUE约束, 当插入出现重复键时就会触发**唯一键冲突 (duplicate key)** , 此时会执行后面的update逻辑

3. 快速忽略重复插入
   ```SQL
   INSERT IGNORE IINTO users(email, name)
   VALUES('example@example.com', 'John, Doe');
   ```

#### Text数据类型可以无限大吗？

MySQL的 三种text类型的最大长度:

- TEXT: 65,535 bytes / 65Kb
- MEDIUMTEXT: 16,777,215 bytes / 16Mb
- LONGTEXT: 4,284,967,295 bytes / 4Gb

