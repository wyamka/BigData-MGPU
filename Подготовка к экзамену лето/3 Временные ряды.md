# БЛОК 1: Ежедневная агрегация поездок
## Выполняется отдельно, возвращает ежедневную статистику
```sql
WITH daily_trips AS (
  SELECT 
    DATE(trip_start_timestamp) AS trip_date,
    COUNT(*) AS trip_count,
    AVG(trip_seconds) AS avg_trip_duration_sec,
    AVG(trip_miles) AS avg_trip_miles,
    SUM(trip_total) AS daily_revenue,
    AVG(trip_total) AS avg_fare,
    AVG(tips) AS avg_tip,
    COUNT(CASE WHEN payment_type = 'Cash' THEN 1 END) AS cash_payments,
    COUNT(CASE WHEN payment_type = 'Credit Card' THEN 1 END) AS card_payments
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
    AND trip_seconds > 0
    AND trip_miles > 0
  GROUP BY trip_date
  ORDER BY trip_date
)

SELECT 
  trip_date,
  trip_count,
  ROUND(avg_trip_duration_sec, 2) AS avg_duration_sec,
  ROUND(avg_trip_miles, 2) AS avg_miles,
  ROUND(daily_revenue, 2) AS daily_revenue,
  ROUND(avg_fare, 2) AS avg_fare,
  ROUND(avg_tip, 2) AS avg_tip,
  cash_payments,
  card_payments
FROM daily_trips
LIMIT 1000;
```

# БЛОК 2: Недельная и месячная статистика для выявления сезонности
```sql
WITH weekly_stats AS (
  SELECT 
    EXTRACT(YEAR FROM trip_start_timestamp) AS year,
    EXTRACT(WEEK FROM trip_start_timestamp) AS week_number,
    COUNT(*) AS trip_count,
    SUM(trip_total) AS weekly_revenue,
    AVG(trip_total) AS avg_fare,
    AVG(trip_miles) AS avg_miles
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
  GROUP BY year, week_number
),

monthly_stats AS (
  SELECT 
    DATE_TRUNC(trip_start_timestamp, MONTH) AS month_start,
    COUNT(*) AS trip_count,
    SUM(trip_total) AS monthly_revenue,
    AVG(trip_total) AS avg_fare,
    COUNT(DISTINCT taxi_id) AS unique_taxis,
    AVG(trip_seconds) AS avg_duration
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
  GROUP BY month_start
)

SELECT 
  'Weekly' AS granularity,
  CAST(year AS STRING) || '-W' || CAST(week_number AS STRING) AS period,
  trip_count,
  ROUND(weekly_revenue, 2) AS revenue,
  ROUND(avg_fare, 2) AS avg_fare,
  ROUND(avg_miles, 2) AS avg_miles
FROM weekly_stats

UNION ALL

SELECT 
  'Monthly' AS granularity,
  CAST(month_start AS STRING) AS period,
  trip_count,
  ROUND(monthly_revenue, 2) AS revenue,
  ROUND(avg_fare, 2) AS avg_fare,
  ROUND(avg_duration, 2) AS avg_miles
FROM monthly_stats
ORDER BY granularity, period;
```


# БЛОК 3: Часовое распределение поездок
```sql
WITH hourly_agg AS (
  SELECT 
    EXTRACT(HOUR FROM trip_start_timestamp) AS hour_of_day,
    COUNT(*) AS trip_count,
    AVG(trip_total) AS avg_fare,
    AVG(trip_miles) AS avg_miles,
    AVG(trip_seconds) AS avg_duration_sec,
    MIN(trip_total) AS min_fare,
    MAX(trip_total) AS max_fare
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
  GROUP BY hour_of_day
),
## Отдельно считаем медиану

hourly_median AS (
  SELECT 
    EXTRACT(HOUR FROM trip_start_timestamp) AS hour_of_day,
    PERCENTILE_CONT(trip_total, 0.5) OVER (PARTITION BY EXTRACT(HOUR FROM trip_start_timestamp)) AS median_fare
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
)
SELECT DISTINCT
  h.hour_of_day,
  h.trip_count,
  ROUND(h.avg_fare, 2) AS avg_fare,
  ROUND(h.avg_miles, 2) AS avg_miles,
  ROUND(h.avg_duration_sec, 0) AS avg_duration_sec,
  ROUND(m.median_fare, 2) AS median_fare,
  ROUND(h.min_fare, 2) AS min_fare,
  ROUND(h.max_fare, 2) AS max_fare
FROM hourly_agg h
LEFT JOIN hourly_median m ON h.hour_of_day = m.hour_of_day
ORDER BY h.hour_of_day;
```


# БЛОК 4: Прогнозирование с помощью скользящего среднего
```sql
WITH daily_series AS (
  SELECT 
    DATE(trip_start_timestamp) AS trip_date,
    COUNT(*) AS actual_count
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
  GROUP BY trip_date
),

full_dates AS (
  SELECT date
  FROM UNNEST(GENERATE_DATE_ARRAY('2023-01-01', '2023-12-31')) AS date
),

daily_complete AS (
  SELECT 
    f.date,
    COALESCE(d.actual_count, 0) AS actual_count
  FROM full_dates f
  LEFT JOIN daily_series d ON f.date = d.trip_date
),

forecast_data AS (
  SELECT 
    date,
    actual_count,
    -- 7-дневное скользящее среднее (включая текущий день)
    AVG(actual_count) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma_7d,
    -- 30-дневное скользящее среднее
    AVG(actual_count) OVER (ORDER BY date ROWS BETWEEN 29 PRECEDING AND CURRENT ROW) AS ma_30d,
    -- Лаги для ES
    LAG(actual_count, 1) OVER (ORDER BY date) AS prev_actual,
    LAG(actual_count, 2) OVER (ORDER BY date) AS prev2_actual
  FROM daily_complete
),

exp_smoothing AS (
  SELECT 
    date,
    actual_count,
    ma_7d,
    ma_30d,
    -- Простое экспоненциальное сглаживание (альфа = 0.3)
    -- Используем предыдущее значение MA(7) как базовое
    0.3 * actual_count + 0.7 * LAG(ma_7d, 1) OVER (ORDER BY date) AS es_forecast
  FROM forecast_data
)
SELECT 
  date,
  actual_count,
  ROUND(ma_7d, 2) AS ma_7d_forecast,
  ROUND(ma_30d, 2) AS ma_30d_forecast,
  ROUND(es_forecast, 2) AS es_forecast,
  ROUND(ABS(actual_count - ma_7d), 2) AS absolute_error_ma7,
  ROUND(POW(actual_count - ma_7d, 2), 2) AS squared_error_ma7,
  ROUND(ABS(actual_count - ma_30d), 2) AS absolute_error_ma30
FROM exp_smoothing
WHERE date >= '2023-07-01'
ORDER BY date
LIMIT 100;
```

# БЛОК 5: Оценка точности прогнозов
```sql
WITH daily_series AS (
  SELECT 
    DATE(trip_start_timestamp) AS trip_date,
    COUNT(*) AS actual_count
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
  GROUP BY trip_date
),

full_dates AS (
  SELECT date
  FROM UNNEST(GENERATE_DATE_ARRAY('2023-01-01', '2023-12-31')) AS date
),

daily_complete AS (
  SELECT 
    f.date,
    COALESCE(d.actual_count, 0) AS actual_count
  FROM full_dates f
  LEFT JOIN daily_series d ON f.date = d.trip_date
),

model_errors AS (
  SELECT 
    date,
    actual_count,
    -- MA(7) прогноз (только предыдущие 7 дней, без текущего)
    AVG(actual_count) OVER (ORDER BY date ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING) AS ma7_pred,
    -- MA(30) прогноз
    AVG(actual_count) OVER (ORDER BY date ROWS BETWEEN 30 PRECEDING AND 1 PRECEDING) AS ma30_pred
  FROM daily_complete
  WHERE date >= '2023-07-01'
),

error_metrics AS (
  SELECT 
    'MA(7)' AS model_name,
    ROUND(AVG(ABS(actual_count - ma7_pred)), 2) AS MAE,
    ROUND(SQRT(AVG(POW(actual_count - ma7_pred, 2))), 2) AS RMSE,
    ROUND(AVG(ABS((actual_count - ma7_pred) / NULLIF(actual_count, 0))) * 100, 2) AS MAPE,
    COUNT(*) AS test_days
  FROM model_errors
  WHERE ma7_pred IS NOT NULL AND actual_count > 0
  
  UNION ALL
  
  SELECT 
    'MA(30)' AS model_name,
    ROUND(AVG(ABS(actual_count - ma30_pred)), 2) AS MAE,
    ROUND(SQRT(AVG(POW(actual_count - ma30_pred, 2))), 2) AS RMSE,
    ROUND(AVG(ABS((actual_count - ma30_pred) / NULLIF(actual_count, 0))) * 100, 2) AS MAPE,
    COUNT(*) AS test_days
  FROM model_errors
  WHERE ma30_pred IS NOT NULL AND actual_count > 0
)

SELECT 
  model_name,
  MAE AS `Mean_Absolute_Error`,
  RMSE AS `Root_Mean_Squared_Error`,
  MAPE AS `MAPE_Percent`,
  test_days AS `Test_Days`
FROM error_metrics
ORDER BY RMSE;
```


# БЛОК 6: Поиск аномалий в поездках
```sql
WITH hourly_stats AS (
  SELECT 
    DATE(trip_start_timestamp) AS trip_date,
    EXTRACT(HOUR FROM trip_start_timestamp) AS hour,
    COUNT(*) AS trip_count,
    AVG(trip_total) AS avg_fare,
    AVG(trip_seconds) AS avg_duration
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
  GROUP BY trip_date, hour
),

hourly_avg_std AS (
  SELECT 
    hour,
    AVG(trip_count) AS avg_count,
    STDDEV(trip_count) AS std_count
  FROM hourly_stats
  GROUP BY hour
),

zscore_calc AS (
  SELECT 
    h.trip_date,
    h.hour,
    h.trip_count,
    h.avg_fare,
    CASE 
      WHEN s.std_count > 0 THEN (h.trip_count - s.avg_count) / s.std_count
      ELSE 0
    END AS zscore_count
  FROM hourly_stats h
  JOIN hourly_avg_std s ON h.hour = s.hour
)

SELECT 
  trip_date,
  hour,
  trip_count,
  ROUND(avg_fare, 2) AS avg_fare,
  ROUND(zscore_count, 2) AS zscore,
  CASE 
    WHEN ABS(zscore_count) > 3 THEN 'Strong Anomaly'
    WHEN ABS(zscore_count) > 2 THEN 'Potential Anomaly'
    ELSE 'Normal'
  END AS anomaly_status
FROM zscore_calc
WHERE ABS(zscore_count) > 2
ORDER BY ABS(zscore_count) DESC
LIMIT 100;
```

# БЛОК 7: Факторный анализ - влияние времени на количество поездок
```sql
SELECT 
  FORMAT_DATE('%A', DATE(trip_start_timestamp)) AS day_of_week,
  FORMAT_DATE('%B', DATE(trip_start_timestamp)) AS month_name,
  EXTRACT(QUARTER FROM trip_start_timestamp) AS quarter,
  EXTRACT(MONTH FROM trip_start_timestamp) AS month_num,
  EXTRACT(DAYOFWEEK FROM trip_start_timestamp) AS day_num,
  COUNT(*) AS trip_count,
  ROUND(AVG(trip_total), 2) AS avg_fare,
  ROUND(AVG(trip_miles), 2) AS avg_miles,
  ROUND(AVG(trip_seconds), 2) AS avg_duration_sec,
  ROUND(AVG(SAFE_DIVIDE(tips, trip_total)) * 100, 2) AS tip_percentage
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_start_timestamp >= '2023-01-01'
  AND trip_start_timestamp < '2024-01-01'
  AND trip_total IS NOT NULL
  AND trip_total > 0
GROUP BY day_of_week, month_name, quarter, month_num, day_num
ORDER BY 
  quarter,
  month_num,
  day_num;
```

# БЛОК 8: Простая линейная регрессия для прогнозирования
```sql
WITH daily_counts AS (
  SELECT 
    DATE(trip_start_timestamp) AS trip_date,
    COUNT(*) AS y
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
  GROUP BY trip_date
),

stats AS (
  SELECT 
    COUNT(*) AS n,
    SUM(EXTRACT(DAYOFYEAR FROM trip_date)) AS sum_x,
    SUM(y) AS sum_y,
    SUM(EXTRACT(DAYOFYEAR FROM trip_date) * y) AS sum_xy,
    SUM(POW(EXTRACT(DAYOFYEAR FROM trip_date), 2)) AS sum_x2
  FROM daily_counts
),

regression AS (
  SELECT 
    SAFE_DIVIDE((n * sum_xy - sum_x * sum_y), (n * sum_x2 - POW(sum_x, 2))) AS slope,
    SAFE_DIVIDE((sum_y - SAFE_DIVIDE((n * sum_xy - sum_x * sum_y), (n * sum_x2 - POW(sum_x, 2))) * sum_x), n) AS intercept
  FROM stats
)

SELECT 
  d.trip_date,
  d.y AS actual_count,
  ROUND(r.intercept + r.slope * EXTRACT(DAYOFYEAR FROM d.trip_date), 2) AS predicted_count,
  ROUND(ABS(d.y - (r.intercept + r.slope * EXTRACT(DAYOFYEAR FROM d.trip_date))), 2) AS abs_error,
  ROUND(SAFE_DIVIDE(ABS(d.y - (r.intercept + r.slope * EXTRACT(DAYOFYEAR FROM d.trip_date))), d.y) * 100, 2) AS error_percentage
FROM daily_counts d
CROSS JOIN regression r
WHERE d.trip_date >= '2023-07-01'
ORDER BY d.trip_date
LIMIT 50;
```

# БЛОК 9: Анализ компаний такси
```sql
SELECT 
  company,
  COUNT(*) AS total_trips,
  SUM(trip_total) AS total_revenue,
  ROUND(AVG(trip_total), 2) AS avg_fare,
  ROUND(AVG(trip_miles), 2) AS avg_miles,
  ROUND(AVG(tips), 2) AS avg_tip,
  ROUND(SAFE_DIVIDE(SUM(tips), SUM(trip_total)) * 100, 2) AS tip_rate
FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
WHERE trip_start_timestamp >= '2023-01-01'
  AND trip_start_timestamp < '2024-01-01'
  AND trip_total IS NOT NULL
  AND company IS NOT NULL
GROUP BY company
ORDER BY total_revenue DESC
LIMIT 10;
```

# БЛОК 10: Сравнение качества моделей на тестовом периоде
```sql
WITH daily_series AS (
  SELECT 
    DATE(trip_start_timestamp) AS trip_date,
    COUNT(*) AS actual_count
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
  WHERE trip_start_timestamp >= '2023-01-01'
    AND trip_start_timestamp < '2024-01-01'
    AND trip_total IS NOT NULL
  GROUP BY trip_date
),

full_dates AS (
  SELECT date
  FROM UNNEST(GENERATE_DATE_ARRAY('2023-01-01', '2023-12-31')) AS date
),

daily_complete AS (
  SELECT 
    f.date,
    COALESCE(d.actual_count, 0) AS actual_count
  FROM full_dates f
  LEFT JOIN daily_series d ON f.date = d.trip_date
),

predictions AS (
  SELECT 
    date,
    actual_count,
    AVG(actual_count) OVER (ORDER BY date ROWS BETWEEN 7 PRECEDING AND 1 PRECEDING) AS ma7,
    AVG(actual_count) OVER (ORDER BY date ROWS BETWEEN 30 PRECEDING AND 1 PRECEDING) AS ma30,
    AVG(actual_count) OVER (ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7_smooth
  FROM daily_complete
  WHERE date >= '2023-10-01'  -- Тестовый период
),

metrics AS (
  SELECT 
    'MA(7)' AS model,
    ROUND(AVG(ABS(actual_count - ma7)), 2) AS MAE,
    ROUND(SQRT(AVG(POW(actual_count - ma7, 2))), 2) AS RMSE,
    ROUND(AVG(ABS(SAFE_DIVIDE((actual_count - ma7), actual_count))) * 100, 2) AS MAPE
  FROM predictions
  WHERE ma7 IS NOT NULL AND actual_count > 0
  
  UNION ALL
  
  SELECT 
    'MA(30)' AS model,
    ROUND(AVG(ABS(actual_count - ma30)), 2) AS MAE,
    ROUND(SQRT(AVG(POW(actual_count - ma30, 2))), 2) AS RMSE,
    ROUND(AVG(ABS(SAFE_DIVIDE((actual_count - ma30), actual_count))) * 100, 2) AS MAPE
  FROM predictions
  WHERE ma30 IS NOT NULL AND actual_count > 0
  
  UNION ALL
  
  SELECT 
    'MA(7) Smooth' AS model,
    ROUND(AVG(ABS(actual_count - ma7_smooth)), 2) AS MAE,
    ROUND(SQRT(AVG(POW(actual_count - ma7_smooth, 2))), 2) AS RMSE,
    ROUND(AVG(ABS(SAFE_DIVIDE((actual_count - ma7_smooth), actual_count))) * 100, 2) AS MAPE
  FROM predictions
  WHERE ma7_smooth IS NOT NULL AND actual_count > 0
)

SELECT 
  model,
  MAE,
  RMSE,
  MAPE AS MAPE_Percent,
  CASE 
    WHEN MAPE < 10 THEN 'Excellent'
    WHEN MAPE < 20 THEN 'Good'
    WHEN MAPE < 30 THEN 'Fair'
    ELSE 'Poor'
  END AS Performance
FROM metrics
ORDER BY RMSE;
```
