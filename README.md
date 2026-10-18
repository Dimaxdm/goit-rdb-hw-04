# HW04: DML та DDL команди. Складні SQL-вирази (Olist + PostgreSQL)

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/188aWEaduA7Z_EMeuP84DcCs0ar20AipX?usp=sharing)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.12%2B-3776AB?logo=python&logoColor=white)
[![Dataset](https://img.shields.io/badge/Dataset-Olist%20Brazilian%20E--commerce-20BEFF?logo=kaggle&logoColor=white)](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)

Домашнє завдання з Теми 4. Багатотабличний e-commerce датасет Olist проходить повний цикл
від сирих CSV до схеми, готової для аналітики й ML:

**raw CSV → typed-схема з constraints → DML (`RETURNING`, `ON CONFLICT`, `UPDATE ... FROM`, `DELETE ... RETURNING`)
→ `customer_segments` на рівні `customer_unique_id` → JOIN-фічі → агрегати → set operations / window functions → Reflection.**

## Структура репозиторію

```
goit-rdb-hw-04/
├── hw04_olist.ipynb     # рішення: notebook з output усіх клітинок
├── requirements.txt     # Python-залежності
└── README.md
```

CSV-файли в репозиторій не комітяться. Notebook завантажує їх сам (див. [Отримання даних](#отримання-даних)).

## Джерело датасету

**Brazilian E-Commerce Public Dataset by Olist**: близько 100 тис. замовлень на бразильській платформі Olist
за 2016–2018 роки. Ліцензія CC BY-NC-SA 4.0.

* Kaggle: <https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce> (`olistbr/brazilian-ecommerce`)

## Отримання даних

Секція **1.2** notebook-а шукає файли відтворювано, у такому порядку:

1. **KaggleHub**: `kagglehub.dataset_download("olistbr/brazilian-ecommerce")`. У Colab датасет береться з кешу Colab (`/kaggle/input/brazilian-ecommerce`), Kaggle credentials не потрібні.
2. **Резерв**: якщо Kaggle недоступний, робиться `git clone` публічного дзеркала оригінальних CSV <https://github.com/spdrio/Brazilian-E-Commerce-Public-Dataset-by-Olist> у `/tmp/olist_mirror`.

Ручне завантаження: скачайте архів зі сторінки Kaggle і розпакуйте 6 файлів, перелічених нижче.
Потім задайте `dataset_dir` на цю папку перед клітинкою 1.3.

> Дзеркало містить старіший реліз Olist (100 000 review-рядків). На ньому кількість відгуків і похідні
> від неї метрики трохи відрізнятимуться від output у notebook, який отримано з версії Kaggle.

## Використані таблиці

| CSV | Raw-таблиця | Typed-таблиця | Ключ у typed-схемі | Рядків |
|---|---|---|---|---:|
| `olist_customers_dataset.csv` | `olist_customers_raw` | `olist_customers` | `customer_id` | 99 441 |
| `olist_orders_dataset.csv` | `olist_orders_raw` | `olist_orders` | `order_id` | 99 441 |
| `olist_order_items_dataset.csv` | `olist_order_items_raw` | `olist_order_items` | `(order_id, order_item_id)` | 112 650 |
| `olist_products_dataset.csv` | `olist_products_raw` | `olist_products` | `product_id` | 32 951 |
| `olist_sellers_dataset.csv` | `olist_sellers_raw` | `olist_sellers` | `seller_id` | 3 095 |
| `olist_order_reviews_dataset.csv` | `olist_order_reviews_raw` | `olist_order_reviews` | технічний `review_row_id` | 99 224 |

Raw-шар повністю переноситься в typed-шар: smoke-test показує 0 втрачених рядків у всіх таблицях.
`review_id` не є primary key, бо 814 review-рядків мають повторний `review_id`. Він зберігається як
джерельний атрибут.

**Похідні та службові таблиці**

| Таблиця | Призначення | Ключ |
|---|---|---|
| `olist_customer_dim` | дедуплікований покупець (96 096 рядків) | `customer_unique_id` |
| `customer_segments` | RFM-сегментація: VIP / regular / new / inactive | identity `segment_id`, `UNIQUE (customer_unique_id)`, FK → `olist_customer_dim` |
| `seller_score` | рейтинг і tier продавця (gold / silver / bronze) | `seller_id` |
| `seller_alerts` | audit-лог alert-ів по продавцях | identity `alert_id` |

## Запуск

### Google Colab (основний спосіб)

1. Відкрийте notebook за кнопкою **Open in Colab** вище (або завантажте `hw04_olist.ipynb` у Colab).
2. **Runtime → Restart session and run all.**

Перша секція виконує такі кроки:

* встановлює PostgreSQL 16 через `apt-get install postgresql`, запускає `service postgresql start` і задає пароль користувачу `postgres`;
* встановлює Python-залежності (`sqlalchemy`, `psycopg2-binary`, `pandas`, `ipython-sql`, `kagglehub`);
* створює SQLAlchemy `engine` (`postgresql+psycopg2://postgres:postgres@localhost/postgres`, `AUTOCOMMIT`) і друкує `SELECT version()`;
* підключає `%sql` / `%%sql` magic до тієї ж бази. Через неї виконуються SQL-запити notebook-а.

Повний прогін займає приблизно 2–3 хвилини, більшість часу йде на `apt-get install`.

### Локально (Ubuntu / Debian)

Потрібен PostgreSQL на `localhost:5432`, користувач `postgres` з паролем `postgres`:

```bash
sudo apt-get install -y postgresql
sudo service postgresql start
sudo -u postgres psql -c "ALTER USER postgres PASSWORD 'postgres';"

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter notebook hw04_olist.ipynb
```

Клітинки з `!apt-get` / `!service` / `!sudo` у першій секції розраховані на Colab. Локально їх можна пропустити, якщо PostgreSQL уже запущено.

### Повторний запуск

Notebook можна запускати повторно без ручного очищення бази:

* typed- і службові таблиці перестворюються через `DROP TABLE IF EXISTS ... CASCADE`;
* raw-таблиці замінюються (`if_exists="replace"`);
* `seller_score` і `customer_segments` наповнюються idempotent upsert-ом (`ON CONFLICT ... DO UPDATE`).

## Зміст notebook-а

| Секція | Що зроблено |
|---|---|
| **1. Setup і typed-схема** | PostgreSQL у Colab, завантаження CSV, raw-шар, typed-таблиці з PK / FK / CHECK / NOT NULL та індексами, smoke-test raw vs typed, перевірка дублікатів `review_id`, перелік constraints |
| **2. DML** | `INSERT ... RETURNING` (alert для продавця з найбільшою кількістю запізнень), `INSERT ... ON CONFLICT DO UPDATE` (`seller_score` на грануляності `seller_id × order_id`, з перевіркою idempotency та доказом завищення відгуків при item-level join), `UPDATE ... FROM` (прапорець `is_late`), `DELETE ... RETURNING` (retention alert-ів) |
| **3. `customer_segments`** | `olist_customer_dim` з `DISTINCT ON`; DDL з identity PK, FK, UNIQUE, CHECK, NOT NULL, DEFAULT; наповнення через `INSERT ... SELECT ... ON CONFLICT` з `reference_date` з датасету; перевірка constraints і negative-тест CHECK |
| **4. JOIN** | `INNER` (4 таблиці), `LEFT` + `COALESCE` (customer-level фічі), `FULL OUTER` (штати клієнтів vs продавців), `SELF` (пари продавців), anti-join (`LEFT JOIN ... IS NULL` і `NOT EXISTS`), а також запізнення vs оцінка. Кожен блок: business-question / SQL / interpretation |
| **5. Агрегати** | revenue за категорією, AOV за штатом, категорії покупця (`STRING_AGG`), профіль low-rating продавців (`JSONB_OBJECT_AGG`, `ARRAY_AGG`); `HAVING` у кількох запитах; пояснення `WHERE` vs `HAVING` |
| **6. Set operations / window** | `UNION`, `INTERSECT`, `EXCEPT`; `ROW_NUMBER()` (останнє замовлення покупця), `RANK()`, `LAG()` |
| **Reflection** | відповіді на 7 запитань: JOIN, DML/DDL-фічі, window functions, data quality, `customer_unique_id`, ризик множення review score |

## Ключові проєктні рішення

* **Грануляність покупця.** Сегментація й customer-фічі рахуються на рівні `customer_unique_id`, а не `customer_id`, бо `customer_id` створюється для кожного замовлення. 99 441 `customer_id` зводяться до 96 096 реальних покупців.
* **Без множення review score.** Позиції спочатку зводяться до `seller_id × order_id`, а відгуки до `order_id`. Наївний join через `order_items` дає 112 372 «відгуки» замість 99 247 і змінює рейтинг 1 264 продавців.
* **Відтворюваність.** `reference_date` для recency береться як `MAX(order_purchase_timestamp) + 1`, а не `CURRENT_DATE`, тому результат не залежить від дати запуску.
