# Практическая работа 2. Изучение и применение различных типов NoSQL баз данных на бизнес-кейсах

**Студентка:** Савкина Мария Алексеевна

**Группа:** БД-251м

**Вариант:** 25

### Введение. Описание бизнес-кейса и выбранного стека технологий.

Цель проекта — создание архитектуры данных на базе принципа Polyglot Persistence, где каждая база данных решает свою задачу:

- MongoDB (Документная СУБД): Хранение каталога контента и профилей. Гибкая структура JSON позволяет оперативно менять метаданные и форматы подписок без остановки системы.

- Cassandra (Wide-Column): Сбор логов и кликстрима. За счет высокой пропускной способности она идеально справляется с интенсивным потоком событий в реальном времени.

- GraphDB (Графовая СУБД): Движок рекомендаций. Эффективно анализирует цепочки связей (актеры, жанры, предпочтения) для формирования точных персональных подборок.

### Архитектура решения

```mermaid
graph TD
    Client[Web / Mobile App] --> API[API Gateway]

    subgraph Polyglot Data Layer
        API -->|1. Каталог контента, Capped Collection для логов| Mongo[(MongoDB Document Store)]
        API -->|2. Логи просмотров| Cass[(Cassandra Wide-Column)]
        API -->|3. SPARQL-запросы, рекомендации| Graph[(GraphDB Graph/RDF)]
    end

    subgraph Analytics Module
        Mongo -.->|Метаданные фильмов, анализ жанров и актёров| BI[Бизнес-Аналитика]
        Cass -.->|Анализ времени просмотра ромкомов| BI
        Graph -.->|Семантические связи, актёры - жанры - фильмы| BI
    end

    classDef db fill:#f9f,stroke:#333,stroke-width:2px;
    class Mongo,Cass,Graph db;

    classDef analytics fill:#e6f7ff,stroke:#1890ff,stroke-width:2px;
    class BI analytics;
```

### Технологический стек

*ОС:* Ubuntu 22.04 (VM Клон devops_dba_25).

*Контейнеризация:* Docker (версия 28.0.0) и Docker Compose (версия v2.33.0). 

*СУБД*: MongoDB 7.0, GraphDB

*Среда разработки:* JupyterLab (Python 3.10), библиотеки pymongo, Faker.

## 2. Развертывание инфраструктуры

![Запуск инфраструктуры](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/Запуск%20инфраструктуры.png)

## Задание.

- MongoDB. capped collections (кольцевые коллекции) для логов.

- Фильмы Tom Hanks и Julia Roberts. Фильмы Nicole Kidman и Tom Cruise до 1980 года.

- Ретроспективный анализ: романтические комедии и их звезды. (С использованием Cassandra)

## 3. Генерация и проверка баз данных (Python + Faker)

### При генерации баз данных зашита бизнес-логика, учтено обязательное наличие фильмов с Tom Hanks и Julia Roberts, а также с Nicole Kidman и Tom Cruise до 1980 года. Отдельно учитаны ромкомы для ретроспективного анализа

```python
!pip install Faker pymongo cassandra-driver

from faker import Faker
from pymongo import MongoClient
from cassandra.cluster import Cluster
import random
import uuid
import time

fake = Faker()

# 1. Инициализация MongoDB
mongo_client = MongoClient("mongodb://root:abc123!@localhost:27017/")
mongo_db = mongo_client["streaming_db"]
movies_col = mongo_db["movies"]
movies_col.drop()

# 2. Инициализация Cassandra
cass_cluster = Cluster(['127.0.0.1'], port=29042, protocol_version=4)
cass_session = cass_cluster.connect()

cass_session.execute("""
    CREATE KEYSPACE IF NOT EXISTS streaming
    WITH replication = {'class':'SimpleStrategy', 'replication_factor':1};
""")
cass_session.set_keyspace('streaming')
cass_session.execute("DROP TABLE IF EXISTS watch_logs;")
cass_session.execute("""
    CREATE TABLE watch_logs (
        movie_id text,
        user_id uuid,
        timestamp timestamp,
        watch_duration_min int,
        PRIMARY KEY (movie_id, timestamp, user_id)
    ) WITH CLUSTERING ORDER BY (timestamp DESC);
""")
print("Успешное подключение к Cassandra!")

# 3. Генерация данных (Mongo + Cassandra + RDF)

# 1. Данные
genres = ["Action", "Comedy", "Drama", "Sci-Fi", "Thriller", "Romance", "Adventure"]

actors_genres = {
    "Tom Hanks": ["Drama", "Comedy"],
    "Julia Roberts": ["Romance", "Drama", "Comedy"],
    "Nicole Kidman": ["Drama", "Thriller", "Romance"],
    "Tom Cruise": ["Action", "Adventure", "Sci-Fi"],
    "Leonardo DiCaprio": ["Drama", "Thriller"],
    "Brad Pitt": ["Action", "Drama"],
    "Scarlett Johansson": ["Action", "Sci-Fi"],
    "Johnny Depp": ["Adventure", "Comedy"],
    "Robert Downey Jr.": ["Action", "Sci-Fi"],
}

actors_list = list(actors_genres.keys())
movie_ids = []

rdf_triples = [
    "@prefix ex: <http://example.org/streaming#> .",
    "@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .\n"
]

# 3. Специальные функции - выбора жанра, количества фильмов по годам
def choose_genres(actors, min_genres=1, max_genres=3):
    possible_genres = set()
    for actor in actors:
        possible_genres.update(actors_genres[actor])
    n = random.randint(min_genres, min(max_genres, len(possible_genres)))
    while len(possible_genres) < n:
        possible_genres.add(random.choice(genres))
    return random.sample(list(possible_genres), n)

mongo_docs = []
romcom_movies = []

def generate_year():
    # экспоненциальный рост индустрии
    r = random.random()
    year = int(1960 + (2026 - 1960) * (r ** 0.5))  
    return year

def romcom_rating_by_year(year):
    # пик 1990
    peak_year = 1990
    # чем дальше от пика - тем ниже рейтинг
    distance = abs(year - peak_year)
    base = 8.5 - (distance / 20)
    
    return max(5.5, min(9.5, base + random.uniform(-0.5, 0.5)))

def general_rating():
    return random.uniform(4.0, 8.5)

# 4. Специальные фильмы
special_movies = [
    {
        "_id": "Movie_1",
        "title": "Tom and Julia Epic",
        "year": generate_year(),
        "genre": choose_genres(["Tom Hanks", "Julia Roberts"]),
        "actors": ["Tom Hanks", "Julia Roberts"],
        "duration_min": random.randint(80, 150),
        "rating": round(random.uniform(3.0, 9.5), 1),
    },
    {
        "_id": "Movie_2",
        "title": "Nicole and Tom Classic",
        "year": random.randint(1960, 1979),
        "genre": choose_genres(["Nicole Kidman", "Tom Cruise"]),
        "actors": ["Nicole Kidman", "Tom Cruise"],
        "duration_min": random.randint(80, 150),
        "rating": round(random.uniform(3.0, 9.5), 1),
    },
]

# 5. Добавляем специальные фильмы
for movie_doc in special_movies:
    mongo_docs.append(movie_doc)
    movie_ids.append(movie_doc["_id"])
    if "Romance" in movie_doc["genre"] and "Comedy" in movie_doc["genre"]:
        romcom_movies.append(movie_doc["_id"])
    
    # RDF
    rdf_triples.append(f'ex:{movie_doc["_id"]} ex:title "{movie_doc["title"]}" ;')
    rdf_triples.append(f'    ex:year "{movie_doc["year"]}"^^xsd:integer ;')
    rdf_triples.append(f'    ex:duration "{movie_doc["duration_min"]}"^^xsd:integer ;')
    for genre in movie_doc["genre"]:
        rdf_triples.append(f'    ex:genre "{genre}" ;')
    for actor in movie_doc["actors"]:
        rdf_triples.append(f'    ex:actor "{actor}" ;')
    rdf_triples.append(f'    ex:rating "{movie_doc["rating"]}"^^xsd:float .\n')

# 6. Генерация 298 случайных фильмов
for i in range(3, 301):
    m_id = f"Movie_{i}"
    movie_ids.append(m_id)
    title = fake.catch_phrase().replace('"', '')
    year = generate_year()
    duration = random.randint(181, 240) if random.choice([True, False]) else random.randint(80, 150)
    actors = random.sample(actors_list, random.randint(2, 4))
    movie_genres = choose_genres(actors)

    is_romcom = "Romance" in movie_genres and "Comedy" in movie_genres

    if is_romcom:
        rating = round(romcom_rating_by_year(year), 1)
    else:
        rating = round(general_rating(), 1)

    movie_doc = {
        "_id": m_id,
        "title": title,
        "year": year,
        "genre": movie_genres,
        "actors": actors,
        "duration_min": duration,
        "rating": rating,
    }

    if "Romance" in movie_genres and "Comedy" in movie_genres:
        romcom_movies.append(m_id)

    mongo_docs.append(movie_doc)

    rdf_triples.append(f'ex:{m_id} ex:title "{title}" ;')
    rdf_triples.append(f'    ex:year "{year}"^^xsd:integer ;')
    rdf_triples.append(f'    ex:duration "{duration}"^^xsd:integer ;')
    for genre in movie_genres:
        rdf_triples.append(f'    ex:genre "{genre}" ;')
    for actor in actors:
        rdf_triples.append(f'    ex:actor "{actor}" ;')
    rdf_triples.append(f'    ex:rating "{movie_doc["rating"]}"^^xsd:float .\n')

movies_col.delete_many({})  # очищаем коллекцию
movies_col.insert_many(mongo_docs)
print(f"Успешно сохранено {len(mongo_docs)} документов в MongoDB.")

# 7. Специальные функции - функция "вероятность досмотра", длительность просмотра 

def completion_probability(movie):
    year = movie["year"]
    genres = movie["genre"]
    base = 0.5  # средний фильм

    if is_romcom:
        if 1980 <= year <= 1999:
            base = 0.8  #  золотая эпоха ромкомов
        else:
            base = 0.6  # обычные ромкомы
    else:
        base = 0.45

    # Влияние рейтинга на досмотр
    base += (movie["rating"] - 6) * 0.05

    return max(0.1, min(0.95, base))

def generate_watch_duration(movie, completed):
    duration = movie["duration_min"]

    if completed:
        return duration

    r = random.random()

    if r < 0.4:
        # бросили в начале
        return int(duration * random.uniform(0.05, 0.2))
    elif r < 0.8:
        # середина
        return int(duration * random.uniform(0.3, 0.7))
    else:
        # почти досмотрели
        return int(duration * random.uniform(0.8, 0.95))

# 8. Генерация логов в Cassandra
for _ in range(5000):
    m_id = random.choice(movie_ids)
    movie = next(m for m in mongo_docs if m["_id"] == m_id)

    u_id = uuid.uuid4()
    ts = fake.date_time_between(start_date='-5y', end_date='now')

    # вероятность досмотра
    p_complete = completion_probability(movie)
    completed = random.random() < p_complete

    watch_dur = generate_watch_duration(movie, completed)

    cass_session.execute(
        """
        INSERT INTO watch_logs (movie_id, user_id, timestamp, watch_duration_min)
        VALUES (%s, %s, %s, %s)
        """,
        (m_id, u_id, ts, watch_dur)
    )

# 8. Сохраняем RDF в файл
with open("movies_graph.ttl", "w", encoding="utf-8") as f:
    f.write("\n".join(rdf_triples))

print(f"RDF файл успешно сгенерирован: movies_graph.ttl")

# ПРОВЕРКА ДАННЫХ В БАЗАХ И ПОЯСНЕНИЕ
print("ПРОВЕРКА СОЗДАННЫХ БАЗ ДАННЫХ")

# MongoDB
mongo_count = movies_col.count_documents({})
sample_movie = movies_col.find_one()

print("--- 1. Проверка MongoDB (Document Store) ---")
print(f"[ДАННЫЕ]: Всего документов (фильмов): {mongo_count}")
print(f"[ДАННЫЕ]: Пример документа:")
print(sample_movie)

print("[ПОЯСНЕНИЕ]: MongoDB успешно сохранила документы фильмов. "
      "Каждый документ содержит название фильма, год выпуска, список актёров, "
      "рейтинг и список жанров. Использование массивов позволяет хранить "
      "несколько жанров для одного фильма (например, ['Romance','Comedy']). "
      "Такая гибкая структура данных удобна для стриминговых сервисов, "
      "где у контента может быть несколько жанров и постоянно добавляются "
      "новые метаданные.\n")

# Cassandra
cass_count = cass_session.execute("SELECT COUNT(*) FROM watch_logs").one()[0]
sample_log = cass_session.execute("SELECT * FROM watch_logs LIMIT 1").one()

print("--- 2. Проверка Cassandra (Wide-Column Store) ---")
print(f"[ДАННЫЕ]: Всего записано логов просмотров: {cass_count}")
print(f"[ДАННЫЕ]: Пример записи:")
print(sample_log)

print("[ПОЯСНЕНИЕ]: Cassandra используется для хранения логов просмотров пользователей. "
      "Каждая запись содержит идентификатор фильма, пользователя, время просмотра "
      "и длительность просмотра. В таблице используется 'movie_id' как ключ партицирования, "
      "а 'timestamp' как ключ кластеризации. Это означает, что логи просмотров одного фильма "
      "физически хранятся вместе и отсортированы по времени, что позволяет эффективно "
      "анализировать популярность контента, например романтических комедий.\n")

# RDF
print("--- 3. RDF граф ---")
print("[ДАННЫЕ]: Файл 'movies_graph.ttl' успешно создан.")

print("[ПОЯСНЕНИЕ]: RDF-граф описывает связи между фильмами, жанрами и актёрами "
      "в виде триплетов (subject–predicate–object). Такая модель позволяет выполнять "
      "семантические запросы (SPARQL) и анализировать связи, например находить "
      "актёров, которые чаще всего снимаются в романтических комедиях.\n")
```

### Скриншоты успешной генерации и проверки

![Генерация](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/Генерация.png)

![Проверка](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/Проверка.png)

**Действие в GraphDB:**
1. Открыт `http://localhost:7200`.
2. Создан репозиторий `movies_repo`.
3. Загружен и импортирован `movies_graph.ttl`.

[movies_graph.ttl](./movies_graph.ttl)

![GraphDB](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/GraphDB.png)

## 4. Задания индивидуального варианта 25.

### 4.1 Capped collections (кольцевые коллекции) для логов.

```python
import threading
import datetime
# Создание capped collection для логов просмотров

print("\n Создание capped collection для логов просмотров")

mongo_db.drop_collection("watch_logs")

mongo_db.create_collection(
    "watch_logs",
    capped=True,
    size=500000,
    max=1000          # максимум 1000 документов
)

logs_col = mongo_db["watch_logs"]

print("[СИСТЕМА]: Capped Collection 'watch_logs' успешно создана.")

print("\n Запись логов просмотров в capped collection")

for _ in range(1500):   # больше чем max, чтобы проверить кольцевой принцип работы
    logs_col.insert_one({
        "movie_id": random.choice(movie_ids),
        "user_id": str(uuid.uuid4()),
        "timestamp": datetime.datetime.now(datetime.UTC),
        "watch_duration_min": random.randint(5, 240)
    })

log_count = logs_col.count_documents({})

print(f"[ДАННЫЕ]: Записано логов после переполнения: {log_count}")

print("\n Проверка capped collection")

sample_log = logs_col.find_one()

print(f"[ДАННЫЕ]: Пример записи лога: {sample_log}")

print("[ПОЯСНЕНИЕ]: Использована capped collection MongoDB — кольцевая структура "
      "фиксированного размера. Когда коллекция достигает лимита (max или size), "
      "самые старые документы автоматически удаляются и заменяются новыми. "
      "Такая модель идеально подходит для хранения потоковых логов просмотров "
      "в стриминговых сервисах.")
```
![Задание1](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/Задание1.png)

### 4.2 Фильмы Tom Hanks и Julia Roberts. Фильмы Nicole Kidman и Tom Cruise до 1980 года. (GraphDB / SPARQL)

*Фильмы Tom Hanks и Julia Roberts.*

```python
import requests
!pip install SPARQLWrapper pandas
from SPARQLWrapper import SPARQLWrapper, JSON
import pandas as pd

# 1. Фильмы Tom Hanks и Julia Roberts

sparql = SPARQLWrapper("http://localhost:17200/repositories/movies_repo")

query1 = """
PREFIX ex: <http://example.org/streaming#>

SELECT ?movie ?title
WHERE {
    ?movie ex:title ?title ;
           ex:actor "Tom Hanks" ;
           ex:actor "Julia Roberts" .
}
"""

sparql.setQuery(query1)
sparql.setReturnFormat(JSON)

print("Поиск фильмов с Tom Hanks и Julia Roberts...")

results = sparql.query().convert()

data_list = []

for result in results["results"]["bindings"]:
    data_list.append({
        "Movie": result["movie"]["value"].split('#')[-1],
        "Title": result["title"]["value"]
    })

df_hanks_roberts = pd.DataFrame(data_list)

print("\n--- Фильмы Tom Hanks и Julia Roberts ---")
print(df_hanks_roberts)
```

![Задание2.1](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/Задание2.1.png)

*Фильмы Nicole Kidman и Tom Cruise до 1980 года.*

```python
# 2. Фильмы Nicole Kidman и Tom Cruise до 1980 года.

query2 = """
PREFIX ex: <http://example.org/streaming#>

SELECT ?movie ?title ?year
WHERE {
    ?movie ex:title ?title ;
           ex:year ?year ;
           ex:actor "Nicole Kidman" ;
           ex:actor "Tom Cruise" .

    FILTER (?year < 1980)
}
"""

sparql.setQuery(query2)
sparql.setReturnFormat(JSON)

print("\nПоиск фильмов Nicole Kidman и Tom Cruise до 1980...")

results = sparql.query().convert()

data_list = []

for result in results["results"]["bindings"]:
    data_list.append({
        "Movie": result["movie"]["value"].split('#')[-1],
        "Title": result["title"]["value"],
        "Year": int(result["year"]["value"])
    })

df_kidman_cruise = pd.DataFrame(data_list)

print("\nФильмы Nicole Kidman и Tom Cruise до 1980")
print(df_kidman_cruise)
```

![Задание2.2](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/Задание2.2.png)

### 4.3 Ретроспективный анализ: романтические комедии и их звезды.

```python
!pip install matplotlib seaborn numpy

# Метрики для ретроспективного анализа
import pandas as pd
from collections import defaultdict, Counter
import numpy as np

df = pd.DataFrame(mongo_docs)

romcoms = df[df["genre"].apply(lambda g: "Romance" in g and "Comedy" in g)]

print(f"Всего ромкомов на платформе: {len(romcoms)}")

# 1. Количество по годам
movies_per_year = romcoms.groupby("year").size()

# 2. Средний рейтинг
avg_rating_per_year = romcoms.groupby("year")["rating"].mean()

# 3. Распределение рейтингов
ratings = romcoms["rating"]

print(f"Средний рейтинг по жанру: {ratings.mean():.2f}")

# 4. Длительность, рейтинг
durations = romcoms["duration_min"]
ratings_scatter = romcoms["rating"]

print(f"Средняя длительность ромкомов: {durations.mean():.1f} мин.")

# 5. Актёры 
actors_exploded = romcoms.explode("actors")

# Топ актёров
top_actors = actors_exploded["actors"].value_counts().head(10)

print("Самые востребованные актеры (Top 5):")
for actor, count in top_actors.head(5).items():
    print(f" - {actor}: {count} фильмов")

# 6. Средний рейтинг актёров
actor_ratings = actors_exploded.groupby("actors")["rating"].mean().sort_values(ascending=False).head(10)

# Добавляем логи из касанды
rows = cass_session.execute("SELECT movie_id, watch_duration_min FROM watch_logs")
logs = pd.DataFrame(list(rows))
logs_df = logs.merge(df, left_on="movie_id", right_on="_id")
romcom_logs = logs_df[logs_df["genre"].apply(lambda g: "Romance" in g and "Comedy" in g)].copy()

print(f"\nВсего логов по ромкомам: {len(romcom_logs)}")

# Доля просмотра
romcom_logs["watch_ratio"] = romcom_logs["watch_duration_min"] / romcom_logs["duration_min"]

# Флаг досмотра
romcom_logs["completed"] = romcom_logs["watch_ratio"] >= 0.9

print(f"Средний процент досмотра: {romcom_logs['watch_ratio'].mean() * 100:.2f}%")
print(f"Рейтинг досмотра ромкомов: {romcom_logs['completed'].mean():.2f}")
```

![Задание3.1](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/Задание3.1.png)

```python
import matplotlib.pyplot as plt
import seaborn as sns

sns.set_theme(style="whitegrid")
plt.rcParams['figure.figsize'] = (10, 6)

# 1. Динамика среднего рейтинга
plt.figure()
sns.lineplot(
    data=romcoms, 
    x="year", 
    y="rating", 
    marker="o", 
    color="darkblue", 
    linewidth=2.5)
plt.title("Динамика качества ромкомов по годам", fontsize=14, fontweight='bold')
plt.xlabel("Год выпуска", fontsize=12)
plt.ylabel("Средний рейтинг", fontsize=12)
plt.ylim(0, 10) 
plt.tight_layout()
plt.show()

# 2. Вероятность досмотра ромкомов по годам
plt.figure()
sns.lineplot(
    data=romcom_logs,
    x="year",
    y="completed",
    estimator="mean",
    linewidth=2
)
plt.title("Вероятность досмотра по годам (RomCom)", fontsize=14, fontweight='bold')
plt.xlabel("Год")
plt.ylabel("Completion rate")
plt.tight_layout()
plt.show()

# 3. Зависимость досмотра ромкома от рейтинга
plt.figure()
sns.lineplot(
    data=romcom_logs,
    x="rating",
    y="completed",
    marker="o",
    linewidth=2
)
plt.title("Зависимость досмотра от рейтинга (RomCom)", fontsize=14, fontweight='bold')
plt.xlabel("Рейтинг фильма")
plt.ylabel("Вероятность досмотра")
plt.ylim(0, 1)
plt.tight_layout()
plt.show()

# 4. Самые востребованные лица 
plt.figure(figsize=(10, 7))
sns.barplot(
    x=top_actors.values, 
    y=top_actors.index,
    hue=top_actors.index,
    palette="mako")
plt.title("Самые востребованные лица", fontsize=14, fontweight='bold')
plt.xlabel("Количество фильмов")
plt.ylabel("") 
plt.tight_layout()
plt.show()

# 5. Средний рейтинг актёров
plt.figure(figsize=(10, 7))
sns.barplot(
    x=actor_ratings.values, 
    y=actor_ratings.index, 
    hue=actor_ratings.index,
    palette="mako")
plt.title("Любимцы критиков (Средний рейтинг)", fontsize=14, fontweight='bold')
plt.xlabel("Средний рейтинг")
plt.ylabel("")

plt.xlim(actor_ratings.min()-0.5, actor_ratings.max()) 

plt.tight_layout()
plt.show()
```

![График1](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/График1.png)

![График2](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/График2.png)

![График3](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/График3.png)

![График4](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/График4.png)

![График5](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/График5.png)

## Бизнес-выводы

```python
# Выводы
business_insights = f"""
Несмотря на небольшое количество (всего  {len(romcoms)} фильмов), жанр показывает очень высокий средний рейтинг {romcoms["rating"].mean():.2f}, то есть  ромкомы явно имеют свою лояльную аудиторию

Самый высокий средний рейтинг у ромкомов "золотой эпохи" {avg_rating_per_year.idxmax()} года - {avg_rating_per_year.max():.2f}, современные фильмы сильно уступают им по качеству
Тренд последних лет показывает, что качество фильмов данного жанра стабильно опускается.

Актёры — драйверы успеха фильма:
Самые востребованные актёры в ромкомах:
{chr(10).join([f"  • {actor}: {count} фильмов" for actor, count in top_actors.head(5).items()])}
То есть Julia Roberts участвует в каждом ромкоме, загруженном на платформу.

Актёры с наивысшим средним рейтингом фильмов:
{chr(10).join([f"  • {actor}: {rating:.2f}" for actor, rating in actor_ratings.head(5).items()])}
Участие этих звёзд заметно коррелирует с более высоким рейтингом.

Всего записей о просмотрах ромкомов: {len(romcom_logs)}.
Средний процент досмотра — {romcom_logs["watch_ratio"].mean() * 100:.1f}%.
Полностью досматривают фильм (≥90%) — {romcom_logs["completed"].mean() * 100:.1f}% зрителей.
Это хороший показатель для комедийного жанра - ромкомы действительно держат внимание до конца.

Ромкомы с рейтингом 8.25-8.60 показывают наилучшее сочетание «нравится большинству» и «досматривают до конца»

Рекомендации для платформы:
1. Активно продвигать ромкомы с участием топ-5 актёров (как по количеству, так и по рейтингу) через карусели, баннеры и персональные подборки.
2. Создавать тематические коллекции «Золотая классика ромкомов» и «Самые недооценённые хиты» с высоким рейтингом и досмотром.
3. Рассмотреть запуск специальных акций / плейлистов в периоды высокого спроса на лёгкий контент (День святого Валентина, новогодние праздники).
"""

print(business_insights)
```
![Выводы](https://raw.githubusercontent.com/wyamka/BigData-MGPU/main/pw_02/screenshots/Выводы.png)

## Вывод
Проект наглядно демонстрирует преимущества полиглотного подхода (Polyglot Persistence) при построении современной архитектуры данных стримингового сервиса. Вместо попытки уместить все типы нагрузок и запросов в одну универсальную базу данных (что почти всегда приводит к компромиссам по производительности, сложности запросов и масштабируемости), каждая подзадача решается наиболее подходящим типом СУБД.