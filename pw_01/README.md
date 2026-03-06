# Практическая работа 1. Обработка и анализ данных с использованием Apache Spark (PySpark).
# Вариант 25
# Выполнила Савкина Мария Алексеевна, группа БД-251м  

## Описание задания варианта:

Платежные системы.

1. Загрузить реестр платежей. Определить долю каждого метода оплаты.

2. Сравнить средний чек (AOV) при оплате картой, наличными и кредитом.

3. Pie chart популярных способов оплаты.

## Источник данных: доработанный дата-сет https://www.kaggle.com/datasets/utkalk/large-retail-data-set-for-eda

[Подробнее об изменении dataset'a](./Dataset)

Вес dataset'a, использованного в работе - 412.48 MB

## Технические характеристики:
ОС: Образ ds_mgpu_Hadoop3+spark_3_4

Hadoop 3.3.5

Spark 3.5.1

Python 3.12.3

JupyterLab

Установленные библиотеки: pyspark, pandas, matplotlib, seaborn, numpy

## Структура проекта
-  [README.md](./README.md)
- Отчет о выполнении проекта [Report.md](./Report.md)
- Основной код решения [pw01_var25.ipynb](./pw01_var25.ipynb)
- Информация о наборе данных [/Dataset](./Dataset/)
- Скриншоты выполнения работы [/Изображения](./Изображения/)
- Файлы, полученные в результате выполнения работы в папке [/output](./output/)

## Инструкция по запуску

1) Запуск hadoop и YARN

```bash
   sudo su - hadoop
   start-dfs.sh
   start-yarn.sh
   jps
   ```

2) Создание директории для работы, загрузка данных в HDFS
```bash
   hdfs dfs -mkdir -p /user/hadoop/pw01_var25
   hdfs dfs -mkdir -p /user/hadoop/pw01_var25/input
   sudo cp /home/devops/Downloads/retail_data_new.csv /tmp/
   sudo chmod 644 /tmp/retail_data_new.csv
   hdfs dfs -put /tmp/retail_data_new.csv /user/hadoop/pw01_var25/input
   hdfs dfs -ls -R /user/hadoop/pw01_var25
```
3. Работа с кодом в JupyterLab
4. Создание директории для вывода результатов работы и предоставление прав
```bash
    hdfs dfs -mkdir -p /user/hadoop/pw01_var25/output
    hdfs dfs -chmod -R 777 /user/hadoop/pw01_var25
```
