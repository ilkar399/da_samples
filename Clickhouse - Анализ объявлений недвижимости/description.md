# Тренировочный проект по Clickhouse - Анализ объявлений недвижимости

* [Исходный датасет на Kaggle](https://www.kaggle.com/code/booroom/russian-real-estate-2021-overview)

## Цели проекта
* Исследование датасета объявлений недвижимости за 2021 год
* Поиск интересных инсайтов в датасете

## Использованные инструменты
* Clickhouse
* CTE
* Оконные функции

## Структура Базы данных

Одна таблица  **russia_real_estate** с 11.358.150 обектов недвижимости в России, собранных из объявлений на avito.ru, realty.yandex.ru, cian.ru, sob.ru, youla.ru, n1.ru, moyareklama.ru за 2021 год.

### Описание полей данных

| Поле          | Тип     | Описание                                                                                         |
| ------------- | ------- | ------------------------------------------------------------------------------------------------ |
| date          | Date    | Дата публикации объявления                                                                       |
| price         | UInt64  | Цена в рублях                                                                                    |
| level         | UInt8   | Этаж                                                                                             |
| levels        | UInt8   | Число этажей                                                                                     |
| rooms         | UInt8   | Число комнат (-1 - студия)                                                                       |
| area          | Float32 | Общая площадь, кв. м                                                                             |
| kitchen_area  | Float32 | Площадь кухни, кв. м                                                                             |
| geo_lat       | Float32 | Широта                                                                                           |
| geo_lon       | Float32 | Долгота                                                                                          |
| building_type | Enum8   | Тип стен (0 - неизвестно, 1 - другое, 2 - панель, 3 - монолит, 4 - кирпич, 5 - блок, 6 - дерево) |
| object_type   | Enum8   | Тип объекта (0 - вторичный рынок, 2 - новострой)                                                 |
| postal_code   | UInt64  | почтовый код                                                                                     |
| street_id     | UInt32  | идентификатор улицы (анонимизированный)                                                          |
| id_region     | UInt8   | идентификатор региона                                                                            |
| house_id      | UInt32  | идентификатор здания (анонимизированный)                                                         |

### Создание Базы данных
```sql
CREATE TABLE russia_real_estate
(
    date Date,
    price UInt64,
    level UInt8,
    levels UInt8,
    rooms Int8,
    area Float32,
    kitchen_area Float32,
    geo_lat Float32,
    geo_lon Float32,
    building_type Enum8('unknown' = 0, 'other' = 1, 'panel' = 2, 'monolithic' = 3, 'brick' = 4, 'blocky' = 5, 'wooden' = 6),
    object_type Enum8('secondary' = 0, 'new' = 2),
    postal_code UInt64,
    street_id UInt32,
    id_region UInt8,
    house_id UInt32
)
ENGINE = MergeTree
ORDER BY (id_region, house_id, geo_lat, geo_lon);
```

### Заполнение Базы Данных
```sql
INSERT INTO russia_real_estate 
SELECT
    parseDateTimeBestEffortUS(date) AS Date,
    toUInt64OrZero(price) AS price,
    toUInt8OrZero(level) AS level,
    toUInt8OrZero(levels) AS levels,
    toInt8OrZero(rooms) AS rooms,
    toFloat32OrZero(area) AS area,
    toFloat32OrZero(kitchen_area) AS kitchen_area,
    toFloat32OrZero(geo_lat) AS geo_lat,
    toFloat32OrZero(geo_lon) AS geo_lon,
    transform(building_type, ['0', '1', '2', '3', '4', '5', '6'], ['unknown', 'other', 'panel', 'monolithic', 'brick', 'blocky', 'wooden']) AS building_type,
    transform(object_type, ['0', '2'], ['secondary', 'new']) AS object_type,
    toUInt64OrZero(postal_code) AS postal_code,
    toUInt32OrZero(street_id) AS street_id,
    toUInt8OrZero(id_region) AS id_region,
    toUInt32OrZero(house_id) AS house_id
FROM input_data;
```
input_data - staging-таблица (может меняться на csv-источник)

## Примеры запросов

### Средняя цена от числа комнат
```sql
SELECT
   rooms AS rooms,
   count(*) as ad_count,
   round(avg(price)) AS avg_price,
   bar(avg_price, 0, 100000000, 80)
FROM russia_real_estate
GROUP BY rooms
ORDER BY rooms
```

### Средняя цена от числа комнат в Москве (точнее, в 77 регионе)
```sql
SELECT
   rooms AS rooms,
   count(*) as ad_count,
   round(avg(price)) AS avg_price,
   bar(avg_price, 0, 250000000, 80)
FROM russia_real_estate
WHERE id_region = 77
GROUP BY rooms
ORDER BY rooms
```

### Места с самыми высокими средними ценами (группировка по региону и почтовому индексу)
```sql
SELECT
    id_region,
    postal_code,
    count() AS c,
    round(avg(price)) AS price,
    bar(price, 0, 250000000, 100)
FROM russia_real_estate
GROUP BY
    id_region,
    postal_code
HAVING c >= 100
ORDER BY price DESC
LIMIT 100
```
