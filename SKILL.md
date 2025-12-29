---
name: psi_rolling_forecast
description: 執行進階的 PSI (進銷存) 滾動預測。處理採購偏移 (+1 month offset) 與期末/期初庫存結轉。
---

# 滾動預測 SOP
當用戶提到「供需缺口」、「未來庫存變化」或「建議採購量」時，使用此技能。

## 核心商務邏輯
- **採購偏移**：`Purchase Forecast` 的庫存需使用 `ADD_MONTHS(..., 1)` 偏移至次月。
- **計算公式**：期末庫存 = 期初庫存 + 供應(t+1) - 需求(t)。
- **輸出規範**：必須遵循標準 9 欄位格式（期間/基準日/期初/需求/供應/月淨變動/期末/狀態/建議採購）。
- **常用模板**：Template 6 (供需缺口), Template 10 (滾動預測), Template 11 (採購建議報告)。

## SQL template
### Template 10: 滾動庫存預測 ⭐⭐⭐
-- 標準格式：所有滾動庫存查詢必須使用此輸出格式
-- Standard Format: All rolling inventory queries MUST use this output format
```sql
-- 用戶問："顯示滾動庫存預測" / "未來庫存變化"
-- 關鍵邏輯：
-- 1. 上期期末庫存 = 本期期初庫存
-- 2. 期初庫存只用 FG + In Transit
-- 3. 庫存基準日取上一個月為當月的庫存基準日
WITH latest_valid_inventory_date AS (
    SELECT
        section,
        SUBSTRING(section, 24) as cutoff_date_str,
        TO_DATE(SUBSTRING(section, 24), 'DD-MON-YY') as cutoff_date
    FROM netsuite.optw_dw_dsi_st
    WHERE section LIKE 'Inventory cut off date:%'
        AND TO_DATE(SUBSTRING(section, 24), 'DD-MON-YY')
            BETWEEN DATE_TRUNC('month', ADD_MONTHS(CURRENT_DATE, -1)::TIMESTAMP)::DATE
            AND LAST_DAY(ADD_MONTHS(CURRENT_DATE, -1))
    ORDER BY cutoff_date DESC
    LIMIT 1
),
current_inventory AS (
    -- 計算期初庫存：只用 FG + In Transit
    SELECT
        SUM(CASE WHEN t.data_type = 'FG + In Transit' THEN t.value ELSE 0 END) as initial_inventory,
        l.cutoff_date_str as inventory_date
    FROM latest_valid_inventory_date l
    JOIN netsuite.optw_dw_dsi_st t
        ON t.section = l.section
        AND t.data_type = 'FG + In Transit'
    GROUP BY l.cutoff_date_str
),
monthly_forecast AS (
    -- Sales Forecast: 當月需求（期間不變）
    SELECT
        SUBSTRING(data_type, 1, 6) as period,
        SUM(value) as demand,
        0 as supply
    FROM netsuite.optw_dw_dsi_st
    WHERE section = 'Sales Forecast'
        AND data_type >= TO_CHAR(CURRENT_DATE, 'YYYYMM')
    GROUP BY SUBSTRING(data_type, 1, 6)

    UNION ALL

    -- Purchase Forecast: 次月供應（+1 month）⭐
    -- 202512 Purchase Forecast → 202601 可用
    SELECT
        TO_CHAR(ADD_MONTHS(TO_DATE(SUBSTRING(data_type, 1, 6), 'YYYYMM'), 1), 'YYYYMM') as period,
        0 as demand,
        SUM(value) as supply
    FROM netsuite.optw_dw_dsi_st
    WHERE section = 'Purchase Forecast'
        AND data_type >= TO_CHAR(CURRENT_DATE, 'YYYYMM')
    GROUP BY TO_CHAR(ADD_MONTHS(TO_DATE(SUBSTRING(data_type, 1, 6), 'YYYYMM'), 1), 'YYYYMM')
),
monthly_forecast_aggregated AS (
    -- 彙總各期間的需求與供應
    SELECT
        period,
        SUM(demand) as demand,
        SUM(supply) as supply
    FROM monthly_forecast
    GROUP BY period
),
forecast_with_cumulative AS (
    SELECT
        period,
        demand,
        supply,
        (supply - demand) as net_change,
        -- 累積淨變動
        SUM(supply - demand) OVER (
            ORDER BY period
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) as cumulative_net
    FROM monthly_forecast_aggregated
)
SELECT
    f.period as 期間,
    i.inventory_date as 庫存基準日,
    -- 期初庫存 = 初始庫存 + 上期累積淨變動（使用 LAG）
    ROUND(
        i.initial_inventory +
        COALESCE(LAG(f.cumulative_net) OVER (ORDER BY f.period), 0),
        0
    ) as 期初庫存,
    ROUND(f.demand, 0) as 需求,
    ROUND(f.supply, 0) as 供應,
    ROUND(f.net_change, 0) as 月淨變動,
    -- 期末庫存 = 初始庫存 + 本期累積淨變動
    ROUND(
        i.initial_inventory + f.cumulative_net,
        0
    ) as 預計期末庫存,
    CASE
        WHEN i.initial_inventory + f.cumulative_net < 0
            THEN '🔴 預計缺貨'
        WHEN i.initial_inventory + f.cumulative_net < 30
            THEN '🟡 低庫存警告'
        WHEN i.initial_inventory + f.cumulative_net < 60
            THEN '🟢 正常'
        ELSE '🟢 健康'
    END as 庫存狀態,
    -- NEW: Recommended Purchase Quantity
    -- 🆕 v1.6 邏輯檢查：當需求為0時，不建議採購（避免庫存累積錯誤）
    CASE
        WHEN i.initial_inventory + f.cumulative_net < 30
            AND f.demand > 0  -- ⭐ 確保未來有需求才建議採購
        THEN ROUND(60 - (i.initial_inventory + f.cumulative_net), 0)
        ELSE NULL
    END as 建議採購量
FROM forecast_with_cumulative f
CROSS JOIN current_inventory i
ORDER BY f.period;
```
