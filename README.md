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

</details>
</details>

<details>
<summary><b>🔥 模块二：直播流量与实时并发分析（Live Stream & PCU Analysis）</b></summary>

<br>

### 题目：直播间历史最大同时在线人数（峰值 PCU）计算
* **业务场景**：**直播运营/内容治理/容量规划**。复盘各场直播流量高峰表现，计算最高同时在线人数（PCU）并筛选 Top3 标杆案例。
* **核心考点**：`断点变迁法 (Event Mark Method)` + `UNION ALL 拆分进入/离开事件` + `窗口累加求和 (SUM() OVER)` + `最大峰值聚合 (MAX)`

<details>
<summary>👉 <b>点击展开查看题目完整描述、输入表结构与 SQL 代码</b></summary>

#### 📋 题目背景
直播运营团队需要复盘各场直播的流量高峰表现。观看日志表记录了用户进出直播间的行为（进入时间、离开时间）。一场直播的“同时在线人数”随用户进出动态变化：用户进入时加 1，离开时减 1。请统计每场直播的历史最大同时在线人数（峰值），并找出峰值最高的前 3 场直播，作为标杆案例在团队内推广。

#### 📋 输入表结构

* **`live_sessions` 表（直播场次表）**
  * `session_id`：直播场次 ID (VARCHAR(20), PRIMARY KEY)
  * `room_name`：直播间名称 (VARCHAR(50))
* **`watch_logs` 表（观看行为日志表）**
  * `log_id`：日志 ID (VARCHAR(20), PRIMARY KEY)
  * `session_id`：直播场次 ID (VARCHAR(20))
  * `user_id`：用户 ID (VARCHAR(20))
  * `enter_time`：进入时间 (DATETIME)
  * `leave_time`：离开时间 (DATETIME)

#### 🎯 输出要求
* **输出字段**：`session_id`（直播场次 ID）、`room_name`（直播间名称）、`peak_concurrent_users`（峰值同时在线人数）。
* **计算逻辑**：
  * **峰值定义**：该场直播过程中任一时刻在线用户数的最大值。
  * **临界点处理**：若某用户离开与另一用户进入发生在同一时刻，该时刻两人视为同时在线（即先算进入 +1，后算离开 -1，或按同一时间戳合并计算）。
  * **筛选规则**：仅输出峰值最高的前 3 场直播。
* **排序规则**：按 `peak_concurrent_users` 降序排列；若峰值相同，按 `session_id` 升序排列。

---

#### 💡 解题思路与断点变迁法
这种“区间重叠 / 最大同时在线”问题的标准解法是**事件拆分与累计求和（Event Marking & Running Total）**：
1. **事件拆分**：把一条日志记录的 `enter_time` 和 `leave_time` 拆成两条独立的“变动事件”。
   * 进入事件：变动值为 `+1`。
   * 离开事件：变动值为 `-1`。
2. **时间戳排序与累加**：按时间戳升序对所有事件进行排序，并使用窗口函数 `SUM(val) OVER(PARTITION BY session_id ORDER BY event_time)` 实时累加计算每个时刻的在线人数。
   * *注意临界点*：题目规定“同一时刻进入与离开视为同时在线”，因此排序时先排 `+1`（进入），再排 `-1`（离开），确保峰值计算不会因先减后加而偏小。
3. **求最大峰值并 TopN 排序**：按 `session_id` 分组取最大在线人数，最后关联直播场次信息取 Top3。

---

#### 答案 SQL 代码

```sql
WITH user_events AS (
    -- 1. 将进入和离开拆分为独立事件，进入为 +1，离开为 -1
    SELECT 
        session_id, 
        enter_time AS event_time, 
        1 AS val
    FROM watch_logs
    
    UNION ALL
    
    SELECT 
        session_id, 
        leave_time AS event_time, 
        -1 AS val
    FROM watch_logs
),
concurrent_stats AS (
    -- 2. 经典窗口函数累加求和：计算每个时间节点的实时在线人数
    -- ORDER BY event_time ASC, val DESC 保证同一时刻先加(+1)后减(-1)
    SELECT 
        session_id,
        SUM(val) OVER (
            PARTITION BY session_id 
            ORDER BY event_time ASC, val DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS current_online
    FROM user_events
),
session_peaks AS (
    -- 3. 按场次分组，取每场直播的历史最大并发人数（峰值）
    SELECT 
        session_id,
        MAX(current_online) AS peak_concurrent_users
    FROM concurrent_stats
    GROUP BY session_id
)
-- 4. 关联直播间信息，取 Top 3 输出
SELECT 
    l.session_id,
    l.room_name,
    p.peak_concurrent_users
FROM session_peaks p
JOIN live_sessions l ON p.session_id = l.session_id
ORDER BY p.peak_concurrent_users DESC, l.session_id ASC
LIMIT 3;
