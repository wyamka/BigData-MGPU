# PySpark
# Выполнила Савкина Мария Алексеевна, группа БД-251м  

## Описание задания:

Работа с HDFS и базовые операции Spark/PySpark: 

Опишите последовательность команд для загрузки текстового или csv файла в HDFS. Напишите псевдокод или основные команды PySpark для выполнения простого анализа этого файла (например, подсчет количества строк, подсчет уникальных слов или фильтрация строк по заданному критерию).

Основные моменты. Команды HDFS, базовые RDD/DataFrame операции в Spark (загрузка, трансформация, действие).

Результаты данных сохранить в HDFS. 

Визуализировать результаты в Yandex DataLens.

Данные: https://drive.google.com/drive/folders/1QiHoIft-v2BOm4Q-xgfR7cvj9kVTHYOc 


## Технические характеристики:
ОС: Образ ds_mgpu_Hadoop3+spark_3_4

Hadoop 3.3.5

Spark 3.5.1

Python 3.12.3

JupyterLab

Установленные библиотеки: pyspark, pandas, matplotlib, seaborn, numpy

## Инструкция по запуску

1) Запуск hadoop и YARN

```bash
   sudo su - hadoop
   #Запуск хранилища
   start-dfs.sh
   #Запуск менеджера ресурсов
   start-yarn.sh
   #Проверка запущенных Java-процессов
   jps
   ```
2) Зайти на http://localhost:9870/ -> утилиты -> browse the file system

Проверить, что запущено и работает

3) Создание директории для работы, загрузка данных в HDFS
```bash
   #Создание папки в HDFS
   hdfs dfs -mkdir -p /user/hadoop/task7
   hdfs dfs -mkdir -p /user/hadoop/task7/input
   #Копирование файла в HDFS
   sudo cp /home/devops/Downloads/brooklyn_sales_map.csv /tmp/
   sudo chmod 644 /tmp/brooklyn_sales_map.csv
   hdfs dfs -put /tmp/brooklyn_sales_map.csv /user/hadoop/task7/input
   #Проверка содержимого папки
   hdfs dfs -ls -R /user/hadoop/task7
```
4) Создание директории для вывода результатов работы и предоставление прав
```bash
    hdfs dfs -mkdir -p /user/hadoop/task1/output
    hdfs dfs -chmod -R 777 /user/hadoop/task7
```

5) PySpark:

### Файл spark.ipynb
```py
!pip install pyspark

# Импорт библиотек
from pyspark.sql import SparkSession
from pyspark.sql.functions import (
    col, count, avg, stddev, hour, to_timestamp,
    regexp_extract, when, lit, window, desc, asc,
    sum as spark_sum,round as spark_round, rand, randn, least,
    date_format
)

from pyspark.sql.types import (
    StructType, StructField, StringType,
    IntegerType, TimestampType, DoubleType
)

#Инициализация SparkSession
# Создание SparkSession
spark = SparkSession.builder \
    .appName("brooklyn_sales_map") \
    .config("spark.hadoop.fs.defaultFS", "hdfs://localhost:9000") \
    .config("spark.ui.port", "4040") \
    .config("spark.sql.shuffle.partitions", "50") \
    .config("spark.driver.memory", "4g") \
    .getOrCreate()

# Установка уровня логирования
spark.sparkContext.setLogLevel("WARN")

print(f"Spark Version: {spark.version}")
print(f"Spark UI: http://localhost:4040")

# Загрузка данных из HDFS в Spark DataFrame
hdfs_path = "hdfs://localhost:9000/user/hadoop/task7/input/brooklyn_sales_map.csv"

df = spark.read \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .csv(hdfs_path)

print("Схема данных brooklyn_sales_map:")
df.printSchema()

print("Общая статистика brooklyn_sales_map:")
print(f"Всего записей: {df.count():,}")

print("\nПримеры данных:")
df.show(5, truncate=False)

#ТРАНСФОРМАЦИЯ
#Фильтрация данных (цена или категория не указаны или маленькая цена)
filtered_df = df.filter(
    col("building_class_category").isNotNull() & 
    col("sale_price").isNotNull() & 
    (col("sale_price") > 10000)
)

#Создание колонки цена за кв фут
df_transformed = filtered_df.withColumn(
    "price_per_sqft",
    when(col("gross_sqft") > 0, col("sale_price") / col("gross_sqft")).otherwise(lit(None))
)

#Агрегация (группировка по районам и категориям зданий)
df_analytics = df_transformed \
    .groupBy("borough1", "building_class_category") \
    .agg(
        count("sale_price").alias("total_sales_count"),
        spark_round(avg("sale_price"), 2).alias("avg_sale_price"),
        spark_round(avg("price_per_sqft"), 2).alias("avg_price_per_sqft")
    ) \
    .orderBy(desc("total_sales_count")) # Сортируем по популярности направлений


# Действие
#Агрегированные результаты
print("Результаты трансформаций (аналитика по районам и категориям зданий):")
df_analytics.show(50, truncate=False)

#Вывод результатов
output_hdfs_path = "hdfs://localhost:9000/user/hadoop/task7/output/brooklyn_sales_map.csv"

print(f"Сохранение результатов в HDFS по пути: {output_hdfs_path}...")
df_analytics.write \
    .mode("overwrite") \
    .parquet(output_hdfs_path)

print("Результаты сохранены в HDFS")

#Сохранение результатов локально:
df_analytics.toPandas().to_csv('/tmp/df_analytics.csv', index=False)

# Остановка SparkSession
spark.stop()
print("SparkSession остановлен")

```   