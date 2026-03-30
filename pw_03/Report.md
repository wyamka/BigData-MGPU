# Практическая работа 3. ML Pipeline на Spark MLlib

> **Вариант 25 (то есть в рамках работы вариант 5)** |

**Цель работы** - построение и обучение модели линейной регрессии, способной предсказывать количество калорий, затрачиваемых пользователем во время физических упражнений, на основе различных признаков активности, таких как длительность тренировки, пульс, скорость движения, перепад высоты и тип спорта.

## Описание задачи

**Бизнес-кейс.** Компания **HealthTech** (аналог Endomondo/Strava).  
Задача — построить прогностическую модель для определения **потраченных калорий** по поведенческим паттернам тренировок (длительность, пульс, скорость, высота и т.д.).

## Задания варианта 5

| № | Задание                                                                                                                        | Реализация                                                                                                               |
| - | ------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------ |
| 1 | **Feature Engineering:** создание признаков `duration` (расчёт), `avg_heart_rate`; синтетическая целевая переменная `calories` | `duration = end_time - start_time`; `avg_heart_rate` как среднее по массиву пульса; `calories = duration * avg_hr / 100` |
| 2 | **Моделирование.** Задача регрессии, модель — линейная регрессия                                                               | Использование Linear Regression для предсказания `calories`                                                              |
| 3 | **Бизнес-метрики.** Оценка качества модели и интерпретация надежности                                                          | Расчёт RMSE и R²; анализ применимости модели для медицинских рекомендаций                                                |


## 0. Setup — Установка зависимостей

```python
import sys
!{sys.executable} -m pip install pyspark gdown findspark matplotlib seaborn scikit-learn --quiet
print('✅ Зависимости установлены')
```
✅ Зависимости установлены

```python
import os, shutil, time, warnings
warnings.filterwarnings('ignore')

import findspark
findspark.init()

import pyspark
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import DoubleType

from pyspark.ml import Pipeline
from pyspark.ml.feature import (StringIndexer, OneHotEncoder,
                                 VectorAssembler, StandardScaler)
from pyspark.ml.classification import RandomForestClassifier
from pyspark.ml.evaluation import (BinaryClassificationEvaluator,
                                    MulticlassClassificationEvaluator)
from pyspark.ml.tuning import CrossValidator, ParamGridBuilder
from pyspark.ml.regression import LinearRegression
import matplotlib.pyplot as plt
import seaborn as sns
import numpy as np
import pandas as pd
from sklearn.metrics import roc_curve, auc, confusion_matrix
from pyspark.ml.evaluation import RegressionEvaluator
from pyspark.ml.linalg import DenseVector

print(f'✅ PySpark: {pyspark.__version__}')
```
✅ PySpark: 4.0.2

## 1. Загрузка данных

```python
DATA_DIR = 'data'
if os.path.exists(DATA_DIR):
    shutil.rmtree(DATA_DIR)
    print(f'🗑️  Каталог {DATA_DIR!r} удалён')
os.makedirs(DATA_DIR)
print(f'📁 Каталог {DATA_DIR!r} создан')
```

🗑️  Каталог 'data' удалён
📁 Каталог 'data' создан

```python
import gdown, zipfile, os

DATA_DIR    = "data"
ZIP_PATH    = os.path.join(DATA_DIR, "endomondoHR.zip")
OUTPUT_PATH = os.path.join(DATA_DIR, "endomondoHR.json")

# Очистка и создание каталога
if os.path.exists(DATA_DIR):
    shutil.rmtree(DATA_DIR)
os.makedirs(DATA_DIR)
print(f"📁 Каталог '{DATA_DIR}' пересоздан")

# Скачивание
FILE_ID = "1yiAp1fFDy3wSqUR0X_btCZPtuczbLwCe"
print("⬇️  Скачивание с Google Drive...")
gdown.download(
    f"https://drive.google.com/uc?id={FILE_ID}&confirm=t",
    ZIP_PATH,
    quiet=False
)
print(f"✅ Скачано: {os.path.getsize(ZIP_PATH)/1024/1024:.1f} МБ")

# Распаковка
print("📦 Распаковка архива...")
with zipfile.ZipFile(ZIP_PATH, "r") as zf:
    print(f"   Содержимое: {zf.namelist()}")
    zf.extractall(DATA_DIR)

# Проверяем что JSON появился
if not os.path.exists(OUTPUT_PATH):
    # иногда внутри архива другое имя — берём первый json
    for f in os.listdir(DATA_DIR):
        if f.endswith(".json"):
            os.rename(os.path.join(DATA_DIR, f), OUTPUT_PATH)
            break

print(f"✅ Готово: {OUTPUT_PATH}")
print(f"   Размер JSON: {os.path.getsize(OUTPUT_PATH)/1024/1024:.1f} МБ")
```

📁 Каталог 'data' пересоздан
⬇️  Скачивание с Google Drive...
Downloading...
From: https://drive.google.com/uc?id=1yiAp1fFDy3wSqUR0X_btCZPtuczbLwCe&confirm=t
To: /content/data/endomondoHR.zip
100%|██████████| 2.02G/2.02G [01:49<00:00, 18.5MB/s]
✅ Скачано: 1926.0 МБ
📦 Распаковка архива...
   Содержимое: ['endomondoHR.json']
✅ Готово: data/endomondoHR.json
   Размер JSON: 6264.1 МБ

## 2. Инициализация SparkSession

```python