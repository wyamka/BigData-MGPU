# PySpark
# Выполнила Савкина Мария Алексеевна, группа БД-251м  

## Описание задания:

Реализовать прогноз данных в PySpark на основании данных файла brooklyn_sales_map.csv, который содержит данные о домах на продажу в Бруклине с 2003 по 2017 года, с использованием логистической регрессии. Добиться прогноза не менее 65%. Визуализировать результаты в Yandex DataLens.

## Источник данных: дата-сет https://drive.google.com/drive/folders/1QiHoIft-v2BOm4Q-xgfR7cvj9kVTHYOc?usp=sharing 

Вес dataset'a, использованного в работе - 197 MB

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
   start-dfs.sh
   start-yarn.sh
   jps
   ```
2) Зайти на http://localhost:9870/ -> утилиты -> browse the file system

Проверить, что запущено и работает

3) Создание директории для работы, загрузка данных в HDFS
```bash
   hdfs dfs -mkdir -p /user/hadoop/task1
   hdfs dfs -mkdir -p /user/hadoop/task1/input
   sudo cp /home/devops/Downloads/brooklyn_sales_map.csv /tmp/
   sudo chmod 644 /tmp/brooklyn_sales_map.csv
   hdfs dfs -put /tmp/brooklyn_sales_map.csv /user/hadoop/task1/input
   hdfs dfs -ls -R /user/hadoop/task1
```
4) Работа с кодом в JupyterLab


// В данной работе не применяется, однако опционально возможно:

5) Создание директории для вывода результатов работы и предоставление прав
```bash
    hdfs dfs -mkdir -p /user/hadoop/task1/output
    hdfs dfs -chmod -R 777 /user/hadoop/task1
```