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
4) PySpark:

5) Создание директории для вывода результатов работы и предоставление прав
```bash
    hdfs dfs -mkdir -p /user/hadoop/task1/output
    hdfs dfs -chmod -R 777 /user/hadoop/task1
```