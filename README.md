# Data-Science-Portfolio
# Data-Science-Portfolio
sql题：
题目描述

背景：
用户运营团队计划开展“沉睡用户唤醒”活动，把历史有过购买、但长期未再消费的用户召回平台。现定义沉睡用户为：累计下单不少于2笔，且最近一笔订单的下单时间距统计基准日 2026-09-01 超过90天的用户。请统计各注册渠道的沉睡用户数量及其平均累计消费金额，用于评估各渠道的唤醒优先级。

输入表结构：
· users 表（全量用户表）：
  · user_id (用户ID)
  · channel (注册渠道)
  · reg_time (注册时间)
· orders 表（订单表）：
  · order_id (订单ID)
  · user_id (用户ID)
  · pay_amount (订单实付金额，DECIMAL类型)
  · order_time (下单时间，DATETIME类型)

输出要求：
· 输出字段：channel（注册渠道）、sleepy_user_cnt（沉睡用户数）、avg_total_amount（平均累计消费金额）。
· 计算逻辑：
  · 沉睡用户：已下单笔数 >= 2，且 MAX(order_time) 距 2026-09-01 超过 90 天。
  · 仅输出沉睡用户数 >= 3 的渠道。
  · 平均累计消费金额 = 该渠道所有沉睡用户的累计实付金额之和 / 沉睡用户数，保留2位小数（四舍五入）。
· 排序规则：按沉睡用户数降序排列，沉睡用户数相同时按注册渠道升序排列。

补充说明：
· TIMESTAMPDIFF(DAY, 开始时间, 结束时间) 用于计算两个日期之间的整天数差值。
· ROUND(数值, 小数位数) 用于四舍五入保留指定小数位数。

```sql
WITH sleepy_users AS (
    -- 步骤1：先筛选出符合“沉睡”标准的用户及其累计消费
    SELECT 
        user_id, 
        SUM(pay_amount) AS total_amount
    FROM orders
    GROUP BY user_id
    HAVING COUNT(order_id) >= 2 
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
