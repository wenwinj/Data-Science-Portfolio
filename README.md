# 📊 Data Science & SQL 核心题解宝典

这里收录了数据科学与数据策略面试中最常见的高频 SQL 场景题，按业务场景进行了模块化分类。

---

## 📂 SQL 题解模块分类

<details>
<summary><b>1️⃣ 模块一：用户生命周期与流失与唤醒分析（User Lifecycle & Churn Analysis）</b></summary>

### 1. 各注册渠道沉睡用户数及平均累计消费金额统计
* **业务背景**：评估沉睡用户（下单 >= 2 笔且 > 90 天未消费）的渠道分布与唤醒优先级。
* **核心考点**：`CTE (WITH 子句)` + `两阶段聚合 (GROUP BY ... HAVING)` + `日期函数 (TIMESTAMPDIFF)`
#### 解题思路
1. **用户粒度筛选 (CTE)**：在 `orders` 表中按 `user_id` 分组，用 `HAVING` 配合 `COUNT(order_id) >= 2` 和 `MAX(order_time)` 筛选出符合条件的沉睡用户，并算出每个用户的累计消费金额。
2. **渠道粒度汇总 (主查询)**：关联 `users` 表获取渠道，按 `u.channel` 分组统计沉睡用户数与平均消费，最后过滤出用户数 `>= 3` 的渠道并排序。

### 题目：
用户运营团队计划开展“沉睡用户唤醒”活动，把历史有过购买、但长期未再消费的用户召回平台。现定义沉睡用户为：累计下单不少于2笔，且最近一笔订单的下单时间距统计基准日 2026-09-01 超过90天的用户。请统计各注册渠道的沉睡用户数量及其平均累计消费金额，用于评估各渠道的唤醒优先级。

<details>
<summary>👉 <b>点击展开查看解题思路与 SQL 代码</b></summary>

### 📋 输入表结构

* **`users` 表（全量用户表）**
  * `user_id`：用户 ID
  * `channel`：注册渠道
  * `reg_time`：注册时间
* **`orders` 表（订单表）**
  * `order_id`：订单 ID
  * `user_id`：用户 ID
  * `pay_amount`：订单实付金额（DECIMAL 类型）
  * `order_time`：下单时间（DATETIME 类型）

### 🎯 输出要求

* **输出字段**：`channel`（注册渠道）、`sleepy_user_cnt`（沉睡用户数）、`avg_total_amount`（平均累计消费金额）
* **计算逻辑**：
  * **沉睡用户定义**：已下单笔数 $\ge 2$，且 `MAX(order_time)` 距 `2026-09-01` 超过 90 天
  * **渠道筛选**：仅输出沉睡用户数 $\ge 3$ 的渠道
  * **金额计算**：平均累计消费金额 = 该渠道所有沉睡用户的累计实付金额之和 / 沉睡用户数，保留 2 位小数（四舍五入）
* **排序规则**：按 `sleepy_user_cnt` 降序排列；若沉睡用户数相同，按 `channel` 升序排列

### 💡 补充说明

* `TIMESTAMPDIFF(DAY, 开始时间, 结束时间)`：用于计算两个日期之间的整天数差值
* `ROUND(数值, 小数位数)`：用于四舍五入保留指定小数位数

sql执行顺序：
1. FROM / JOIN      ← 先找表
2. WHERE            ← 过滤“行”（聚合前）
3. GROUP BY         ← 分组
4. 聚合函数         ← COUNT / SUM / AVG / STDDEV
5. HAVING           ← 过滤“组”（聚合后）
6. SELECT           ← 选列
7. ORDER BY         ← 排序

```sql
WITH sleepy_users AS (
    -- 步骤1：先筛选出符合“沉睡”标准的用户及其累计消费
    SELECT 
        user_id, 
        SUM(pay_amount) AS total_amount
    FROM orders
    GROUP BY user_id
    HAVING COUNT(order_id) >= 2 --having聚合函数
       AND TIMESTAMPDIFF(DAY, MAX(order_time), '2026-09-01') > 90
)
-- 步骤2：关联用户表，按注册渠道进行二次聚合
SELECT
    u.channel,
    COUNT(s.user_id) AS sleepy_user_cnt,
    ROUND(SUM(s.total_amount) / COUNT(s.user_id), 2) AS avg_total_amount
FROM sleepy_users s
JOIN users u ON s.user_id = u.user_id
GROUP BY u.channel
HAVING COUNT(s.user_id) >= 3
ORDER BY sleepy_user_cnt DESC, u.channel ASC;
```
