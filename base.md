---
name: base information
description: DSI base logic and table schema。
---


## Database Schema
```sql
TABLE: netsuite.optw_dw_dsi_st
- region (VARCHAR)          -- 地區代碼 (e.g., 'AP')
- primary_model (VARCHAR)     -- 主model (e.g., 'CinemaX D2')
- secondary_model  (VARCHAR)   -- 次model  
- dmd_chp (DECIMAL)         -- 需求參數 (e.g., 0.47)
- section (VARCHAR)         -- 資料類別（部分包含日期）
- data_type (VARCHAR)       -- 時間週期或庫存分類
- value (DECIMAL)           -- 數值
```

## Section Types
- `TOTAL SALES` - 實際銷售
- `Sales Forecast` - 銷售預測
- `Purchase Forecast` - 採購預測
- `Inventory cut off date: DD-MMM-YY` - 庫存快照（含日期）⭐
- `Delivery Plan` - 交貨計劃

## Data Type Formats
**時間型（用於銷售/預測）：**
- `YYYYMM` - 月份 (202401, 202512)
- `YYYYMM Sales Forecast` - 銷售預測期間
- `YYYYMM Purchase Forecast` - 採購預測期間

**庫存型（用於庫存查詢）：**
- `FG` - 成品庫存
- `FG + In Transit` - 成品+在途（完整可用庫存）⭐
- `LII` - 本地庫存指標

---

## 🎯 Critical Business Logic

### 期初庫存計算（核心邏輯）⭐⭐⭐

**公式：**
```
期初庫存 = FG + In Transit
```

**重要：**
- 只使用 `FG + In Transit` 的值
- `FG + In Transit` 已經包含完整的可用庫存

**SQL 實現：**
```sql
SUM(CASE WHEN data_type = 'FG + In Transit' THEN value ELSE 0 END)
```

### Purchase Forecast 可用月份偏移（核心邏輯）⭐⭐⭐

**規則：**
- Purchase Forecast 的庫存在下個月才可用
- 202512 Purchase Forecast → 202601 可用
- 需使用 ADD_MONTHS() 進行月份偏移

**原因：**
- 採購預測代表當月下單，實際到貨是下個月
- 因此需要將期間往後推一個月

**SQL 實現：**
```sql
TO_CHAR(ADD_MONTHS(TO_DATE(SUBSTRING(data_type, 1, 6), 'YYYYMM'), 1), 'YYYYMM')
```

**範例：**
```sql
-- 將 Purchase Forecast 期間轉換為可用月份
SELECT
    SUBSTRING(data_type, 1, 6) as 原始期間,
    TO_CHAR(ADD_MONTHS(TO_DATE(SUBSTRING(data_type, 1, 6), 'YYYYMM'), 1), 'YYYYMM') as 可用月份
FROM netsuite.optw_dw_dsi_st
WHERE section = 'Purchase Forecast'
-- 結果：202512 → 202601
```

### 庫存基準日選擇（核心邏輯）⭐⭐⭐

**規則：**
- 取上一個月為當月的庫存基準日
- 確保使用完整月份的庫存數據
- 使用日期比較排序，不使用字串 MAX()

**SQL 實現：**
```sql
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
)
```
