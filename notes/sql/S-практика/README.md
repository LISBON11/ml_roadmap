# Практика S — Рабочий SQL для LLM-приложений

Набор заданий под **блок S** дорожной карты: базовый рабочий SQL (S.1–S.6) + то, чего нет ни в одном обычном курсе и что связывает SQL с твоим RAG — **pgvector (S.7 ⭐)** и **text-to-SQL как инструмент агента (S.8)**. Данные — как в реальной работе AI Engineer: логи прогонов LLM, эмбеддинги чанков, метаданные документов.

> **Проверка по темам живёт здесь, а не в карте.** В `roadmap/10` описаны только сами темы (S.1–S.8) — все задания на «докажи себе» собраны в этот файл и в [практику A](../практика/README.md). Прошла блок SQL целиком → прогоняешь эти задачи как единую проверку.

> **Как это соотносится с [практикой A](../практика/README.md):**
> - **S-практика (этот файл)** — рабочий SQL + LLM-специфика: CTE, изменение данных/схема, **векторный поиск**, безопасность text-to-SQL. Проходишь **первой**.
> - **A-практика** — аналитический SQL из спринтов Практикума (множества, `CASE`, даты, **оконные функции**, перцентили, ad-hoc). Проходишь **после**.
> - **Финальный проект темы** — капстоун, собирающий S+A вместе, лежит в [отдельном файле](../финал/README.md).
>
> Базовые выборки/джойны (S.1–S.4) живут **здесь**, чтобы не дублироваться. В A-практике их больше нет — она сразу начинается с аналитики.

---

## Песочница: где запускать

Основы (S.1–S.6) — в любом онлайн-раннере Postgres. **pgvector (S.7) требует расширения `vector`, которого нет в DB Fiddle** — для него нужен Neon/Supabase или локальный Postgres.

1. **S.1–S.6 → [DB Fiddle](https://www.db-fiddle.com/)** *(проверено на июль 2026).* Слева выбери **PostgreSQL**, вставь seed-схему ниже в «Schema SQL», запросы — справа, жми Run.
2. **S.7 (pgvector) → [Neon](https://neon.tech/) или [Supabase](https://supabase.com/)** *(бесплатный tier, поддерживают `create extension vector`).* Один раз заливаешь seed через их веб-SQL-редактор — и база живёт. Это же ляжет прямым мостом в Этап 4 (RAG): позже подключишься к ней из Python.
   - Локальная альтернатива: `docker run -p 5432:5432 -e POSTGRES_PASSWORD=pg pgvector/pgvector:pg16` — образ Postgres с уже собранным pgvector.
3. **Добить технику на нейтральных данных — [PGExercises](https://pgexercises.com/)** *(бесплатно).* Разделы select → joins → aggregates отшлифуют синтаксис S.1–S.4.

---

## Seed-схема (основная — скопируй в песочницу один раз)

Та же схема, что и в A-практике (логи LLM), чтобы навык переносился между блоками.

<details>
<summary>📦 Показать SQL для создания и наполнения таблиц</summary>

```sql
-- Модели (справочник)
CREATE TABLE models (
  model_id   int PRIMARY KEY,
  name       text,
  provider   text,
  price_in   numeric,  -- $ за 1К входных токенов
  price_out  numeric   -- $ за 1К выходных токенов
);
INSERT INTO models VALUES
 (1,'gpt-x','OpenAI',0.005,0.015),
 (2,'claude-y','Anthropic',0.003,0.015),
 (3,'gemini-z','Google',0.002,0.010),
 (4,'local-llama','Local',0.000,0.000);

-- Версии промптов
CREATE TABLE prompt_versions (
  pv_id      int PRIMARY KEY,
  prompt     text,
  version    int,
  created_at timestamp
);
INSERT INTO prompt_versions VALUES
 (1,'support_rag',1,'2026-06-01 10:00'),
 (2,'support_rag',2,'2026-06-10 12:00'),
 (3,'support_rag',3,'2026-06-20 09:00'),
 (4,'classifier',1,'2026-06-05 08:00'),
 (5,'classifier',2,'2026-06-18 15:00');

-- Прогоны LLM
CREATE TABLE runs (
  run_id     int PRIMARY KEY,
  model_id   int REFERENCES models,
  pv_id      int REFERENCES prompt_versions,
  created_at timestamp,
  tokens_in  int,
  tokens_out int,
  latency_ms int,
  status     text  -- 'ok' | 'error' | 'timeout'
);
INSERT INTO runs VALUES
 (1,1,1,'2026-06-21 09:01',1200,300,1800,'ok'),
 (2,2,1,'2026-06-21 09:05',1100,280,1500,'ok'),
 (3,1,2,'2026-06-21 10:00',1300,320,2100,'ok'),
 (4,3,2,'2026-06-21 10:10', 900,250, 900,'ok'),
 (5,1,2,'2026-06-22 11:00',1250,310,2500,'timeout'),
 (6,2,3,'2026-06-22 12:00',1150,290,1400,'ok'),
 (7,3,3,'2026-06-23 08:30', 950,260,1000,'ok'),
 (8,1,3,'2026-06-23 09:00',1400,350,2300,'ok'),
 (9,2,3,'2026-06-24 14:00',1000,270,1300,'ok'),
 (10,4,3,'2026-06-24 14:30', 800,200,4000,'ok'),
 (11,1,4,'2026-06-25 09:00', 500,100, 700,'ok'),
 (12,2,4,'2026-06-25 09:10', 480, 90, 650,'ok'),
 (13,3,5,'2026-06-26 10:00', 520,110, 600,'error'),
 (14,1,5,'2026-06-26 10:20', 510,105, 720,'ok'),
 (15,2,5,'2026-06-27 11:00', 495, 95, 640,'ok');

-- Метрики eval (по прогонам) — заметь: eval есть НЕ у каждого прогона (будут пропуски)
CREATE TABLE evals (
  eval_id  int PRIMARY KEY,
  run_id   int REFERENCES runs,
  metric   text,   -- 'faithfulness' | 'relevance' | 'accuracy'
  score    numeric -- 0..1
);
INSERT INTO evals VALUES
 (1,1,'faithfulness',0.82),(2,1,'relevance',0.90),
 (3,2,'faithfulness',0.88),(4,2,'relevance',0.85),
 (5,3,'faithfulness',0.75),(6,4,'relevance',0.92),
 (7,6,'faithfulness',0.91),(8,7,'relevance',0.87),
 (9,8,'faithfulness',0.79),(10,9,'faithfulness',0.93),
 (11,11,'accuracy',0.94),(12,12,'accuracy',0.90),
 (13,14,'accuracy',0.88),(14,15,'accuracy',0.95);
```
</details>

---

## Задания (от простого к сложному)

Каждое — вопрос, который реально задаёт себе AI-инженер. Реши сам, потом раскрой решение для сверки. Цель — покрыть **все функции S.1–S.8**, а не «нарешать много».

### Выборка и фильтрация (S.1–S.2)
**1.** Выведи 10 прогонов, отсортированных по латентности убыванием; только `run_id`, `model_id`, `latency_ms`, `status`.
<details><summary>решение</summary>

```sql
SELECT run_id, model_id, latency_ms, status
FROM runs
ORDER BY latency_ms DESC
LIMIT 10;
```
Покрывает: `SELECT`, `ORDER BY`, `LIMIT`. Прямая аналогия с `df[[...]].sort_values(...).head(10)` из Этапа 0.
</details>

**2.** Найди прогоны, где `tokens_in` между 1000 и 1300 **и** промпт относится к `support_rag` (pv_id 1–3), исключив статус `ok`. Используй `BETWEEN`, `IN`, `<>`.
<details><summary>решение</summary>

```sql
SELECT run_id, tokens_in, pv_id, status
FROM runs
WHERE tokens_in BETWEEN 1000 AND 1300
  AND pv_id IN (1,2,3)
  AND status <> 'ok';
```
</details>

### Пропуски и чистка данных (S.2)
**3.** У части прогонов нет оценки `faithfulness`. Выведи для каждого прогона его faithfulness, подставив `0` там, где оценки нет (`LEFT JOIN evals` + `COALESCE`). Заодно посчитай отношение `tokens_out / tokens_in`, защитив деление от нуля через `NULLIF`.
<details><summary>решение</summary>

```sql
SELECT r.run_id,
       COALESCE(e.score, 0) AS faithfulness,                          -- пропуск → 0
       ROUND(r.tokens_out::numeric / NULLIF(r.tokens_in, 0), 3) AS out_in_ratio
FROM runs r
LEFT JOIN evals e ON e.run_id = r.run_id AND e.metric = 'faithfulness';
```
`COALESCE(x, 0)` — **заполнение пропуска**: первый не-`NULL` аргумент (тут `NULL` → `0`). `NULLIF(a, b)` — обратное: превращает `a` в `NULL`, если `a = b`; так `NULLIF(tokens_in, 0)` глушит деление на ноль (в SQL деление на `NULL` даёт `NULL`, а не ошибку). Это базовая чистка данных под датасет/фичи — в Практикуме это блок «работа с пропусками».
</details>

### Агрегация (S.3)
**4.** Для каждой модели: число прогонов, средняя и максимальная латентность. Оставь только модели с ≥ 3 прогонами (`HAVING`).
<details><summary>решение</summary>

```sql
SELECT model_id, COUNT(*) AS runs, AVG(latency_ms) AS avg_lat, MAX(latency_ms) AS max_lat
FROM runs
GROUP BY model_id
HAVING COUNT(*) >= 3;
```
Это `groupby(...).agg(...)` из Этапа 0 на языке SQL. `HAVING` фильтрует **после** агрегации (в отличие от `WHERE`).
</details>

### Соединения (S.4)
**5.** Замени `model_id` на имя модели: средняя латентность по имени (INNER JOIN с `models`), по возрастанию.
<details><summary>решение</summary>

```sql
SELECT m.name, AVG(r.latency_ms) AS avg_lat
FROM runs r
JOIN models m ON m.model_id = r.model_id
GROUP BY m.name
ORDER BY avg_lat;
```
</details>

**6.** Найди модели, у которых **нет ни одного прогона** (LEFT JOIN + `IS NULL`).
<details><summary>решение</summary>

```sql
SELECT m.name
FROM models m
LEFT JOIN runs r ON r.model_id = m.model_id
WHERE r.run_id IS NULL;
```
Ловушка: «нет пары» ловится через `IS NULL` по ключу правой таблицы, а не обычным `WHERE` по её полю (иначе `LEFT` схлопнется в `INNER`).
</details>

**7.** Перепиши №6 через `RIGHT JOIN`, поменяв таблицы местами. Затем через `FULL JOIN` выведи разом «висячие» строки с обеих сторон (и модели без прогонов, и прогоны с несуществующей моделью). Когда `FULL` реально нужен?
<details><summary>решение</summary>

```sql
-- RIGHT JOIN = зеркало LEFT из №6: ведущая теперь правая таблица
SELECT m.name
FROM runs r
RIGHT JOIN models m ON m.model_id = r.model_id
WHERE r.run_id IS NULL;

-- FULL JOIN: строки без пары СРАЗУ с обеих сторон
SELECT m.name AS model, r.run_id
FROM models m
FULL JOIN runs r ON r.model_id = m.model_id
WHERE m.model_id IS NULL OR r.run_id IS NULL;
```
`RIGHT` — та же логика, что `LEFT`, но ведущая таблица справа (на практике почти всегда пишут `LEFT`, читается ровнее). `FULL` = `LEFT` ∪ `RIGHT`: показывает несопоставленные строки обеих таблиц. На чистых данных вторая выборка пуста — `FK` гарантирует, что у каждого прогона есть модель; `FULL` нужен, когда таблицы связаны слабо (сырьё из внешних источников до чистки). ⚠️ `JOIN` без `ON` = декартово произведение (все пары со всеми).
</details>

### Подзапросы и CTE (S.5)
**8.** Выведи модели, чья средняя латентность **выше глобальной средней по всем прогонам**. Реши подзапросом в `HAVING`.
<details><summary>решение</summary>

```sql
SELECT m.name, AVG(r.latency_ms) AS avg_lat
FROM runs r
JOIN models m ON m.model_id = r.model_id
GROUP BY m.name
HAVING AVG(r.latency_ms) > (SELECT AVG(latency_ms) FROM runs);
```
Скалярный подзапрос `(SELECT AVG...)` считает один порог, с которым сравнивается каждая группа.
</details>

**9.** Посчитай **стоимость каждого прогона** и оформи её как CTE `costs`, затем из этого CTE выведи топ-5 самых дорогих с именем модели. Формула: `tokens_in/1000*price_in + tokens_out/1000*price_out`.
<details><summary>решение</summary>

```sql
WITH costs AS (
  SELECT r.run_id, m.name,
         ROUND(r.tokens_in/1000.0*m.price_in + r.tokens_out/1000.0*m.price_out, 4) AS cost
  FROM runs r
  JOIN models m ON m.model_id = r.model_id
)
SELECT * FROM costs ORDER BY cost DESC LIMIT 5;
```
CTE (`WITH ... AS`) — твой главный инструмент читаемости: считаешь стоимость один раз, дальше работаешь с ней как с таблицей. Эта же `costs` понадобится в A-практике (NTILE по стоимости, ad-hoc). Задача — прямой аналог cost-оптимизации из Этапа 5.1.5.
</details>

### Изменение данных, схема и витрины (S.6)
**10.** Заведи таблицу разметки ответов `annotations` (id — первичный ключ, ссылка на прогон, метка, комментарий, дата). Вставь 3 строки, обнови метку одной, удали одну.
<details><summary>решение</summary>

```sql
CREATE TABLE annotations (
  ann_id     int PRIMARY KEY,
  run_id     int REFERENCES runs,           -- FOREIGN KEY: связь 1:N с runs
  label      text,          -- 'good' | 'bad' | 'needs_review'
  note       text,
  created_at timestamp
);

INSERT INTO annotations VALUES
 (1, 1, 'good',         'точный ответ',        '2026-06-21 12:00'),
 (2, 5, 'bad',          'таймаут, нет ответа', '2026-06-22 12:00'),
 (3, 8, 'needs_review', 'спорная выдача',      '2026-06-23 12:00');

UPDATE annotations SET label = 'good' WHERE ann_id = 3;   -- пересмотрели
DELETE FROM annotations WHERE ann_id = 2;                 -- убрали шум
```
`PRIMARY KEY` = уникальность + автоиндекс. `REFERENCES runs` — внешний ключ (`FOREIGN KEY`): реализует связь **1:N** (один прогон → много разметок) и не даст вставить разметку на несуществующий прогон. Это и есть «связи между таблицами» из ER-диаграммы.
</details>

**11.** Индексы — на уровне идеи. Какой индекс ускорит частый запрос «все прогоны одной модели за период» (`WHERE model_id = ? AND created_at BETWEEN ? AND ?`), и почему он тут помогает?
<details><summary>решение</summary>

```sql
CREATE INDEX idx_runs_model_time ON runs (model_id, created_at);
```
Идея: индекс — как алфавитный указатель в книге. Без него Postgres читает всю таблицу (seq scan); с индексом по `(model_id, created_at)` он сразу прыгает к нужной модели и диапазону дат. Порядок колонок важен: сначала точное равенство (`model_id`), потом диапазон (`created_at`). Глубже (планы запросов, покрывающие индексы) — территория DBA, тебе не нужна.
</details>

**12.** Собери **витрину данных** `run_costs` как `VIEW` над `runs`+`models` (run_id, имя модели, статус, стоимость), затем читай из неё как из обычной таблицы — топ-5 дорогих. Бонус: посмотри её колонки через `information_schema`.
<details><summary>решение</summary>

```sql
CREATE VIEW run_costs AS
SELECT r.run_id, m.name AS model, r.status,
       ROUND(r.tokens_in/1000.0*m.price_in + r.tokens_out/1000.0*m.price_out, 4) AS cost
FROM runs r
JOIN models m ON m.model_id = r.model_id;

-- витрина используется как обычная таблица
SELECT * FROM run_costs ORDER BY cost DESC LIMIT 5;

-- знакомство со схемой: какие колонки у витрины?
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'run_costs';
```
`VIEW` — сохранённый запрос («витрина»): аналитики читают готовый срез, не переписывая `JOIN` каждый раз. Данные не копируются — вью считается на лету при обращении. `information_schema` — системный каталог: так исследуют **незнакомую** базу («что тут вообще за таблицы и колонки»).
</details>

### PostgreSQL + pgvector (S.7 ⭐) — ядро блока
> **Запускай на Neon/Supabase/локальном Postgres с pgvector.** Векторы намеренно 3-мерные — чтобы косинусную близость можно было прикинуть в уме (это ровно та `cosine` из Этапа 1.1.3, только на проде).

Отдельный seed для этого раздела:

<details>
<summary>📦 Seed для pgvector (чанки документов с эмбеддингами)</summary>

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE doc_chunks (
  chunk_id  int PRIMARY KEY,
  doc       text,
  category  text,        -- 'faq' | 'guide' | 'legal'
  content   text,
  embedding vector(3)    -- в реале 768/1536; тут 3 для наглядности
);

INSERT INTO doc_chunks VALUES
 (1,'refunds','faq',   'Как оформить возврат средств',   '[0.90, 0.10, 0.00]'),
 (2,'refunds','faq',   'Сроки зачисления возврата',      '[0.80, 0.20, 0.05]'),
 (3,'setup',  'guide', 'Установка приложения',           '[0.10, 0.90, 0.15]'),
 (4,'setup',  'guide', 'Настройка профиля пользователя', '[0.15, 0.85, 0.10]'),
 (5,'terms',  'legal', 'Условия использования сервиса',  '[0.05, 0.10, 0.95]'),
 (6,'privacy','legal', 'Политика конфиденциальности',    '[0.05, 0.20, 0.90]');
```
</details>

**13.** Создай индекс **HNSW** под косинусную близость на колонке `embedding`.
<details><summary>решение</summary>

```sql
CREATE INDEX ON doc_chunks USING hnsw (embedding vector_cosine_ops);
```
`hnsw` — приближённый поиск ближайших соседей (быстрый на больших объёмах). `vector_cosine_ops` привязывает индекс к оператору `<=>` (косинус). Под L2 (`<->`) был бы `vector_l2_ops`.
</details>

**14.** Запрос-заглушка пришёл как эмбеддинг `[0.85, 0.15, 0.05]`. Выведи **топ-3 ближайших по косинусу** чанка и саму дистанцию.
<details><summary>решение</summary>

```sql
SELECT chunk_id, doc, content,
       embedding <=> '[0.85, 0.15, 0.05]' AS distance
FROM doc_chunks
ORDER BY embedding <=> '[0.85, 0.15, 0.05]'
LIMIT 3;
```
`<=>` — косинусная дистанция (меньше = ближе). Топ-K = `ORDER BY <дистанция> LIMIT k`. Ближайшими окажутся два `refunds`-чанка — их вектор «смотрит» в ту же сторону, что и запрос.
</details>

**15. ⭐ Гибридный поиск — главный паттерн RAG.** Верни **топ-2 ближайших по косинусу**, но **только среди категории `faq`** (векторный поиск + фильтр по метаданным в одном запросе).
<details><summary>решение</summary>

```sql
SELECT chunk_id, content, embedding <=> '[0.85, 0.15, 0.05]' AS distance
FROM doc_chunks
WHERE category = 'faq'
ORDER BY embedding <=> '[0.85, 0.15, 0.05]'
LIMIT 2;
```
Это ровно тот «семантический поиск с фильтрацией», который в Этапах 1 и 4.3 ты делала руками на NumPy — только теперь один SQL-запрос на проде. `WHERE` по метаданным + `ORDER BY <дистанция>` = сердце retrieval в pgvector-RAG.
</details>

**16.** Сравни метрики: для того же запроса `[0.85, 0.15, 0.05]` выведи рядом косинусную (`<=>`) и L2 (`<->`) дистанцию для каждого чанка. Совпадает ли порядок ближайших?
<details><summary>решение</summary>

```sql
SELECT chunk_id, doc,
       embedding <=> '[0.85, 0.15, 0.05]' AS cosine_dist,
       embedding <-> '[0.85, 0.15, 0.05]' AS l2_dist
FROM doc_chunks
ORDER BY cosine_dist;
```
Косинус смотрит на **направление** вектора (норма не важна), L2 — на **абсолютное расстояние** в пространстве. Для нормализованных эмбеддингов порядок обычно совпадает; на ненормализованных может разойтись. В RAG почти всегда берут косинус — отсюда `<=>` и `vector_cosine_ops` в индексе.
</details>

### Text-to-SQL как агентный паттерн (S.8)
**17.** Агенту дали инструмент «сходи в базу»: модель генерирует SQL по вопросу пользователя, твой код исполняет. Модель сгенерировала:
```sql
DELETE FROM runs WHERE status = 'error'; DROP TABLE feedback;
```
Объясни, почему это исполнять нельзя, и перечисли конкретные ограждения.
<details><summary>решение</summary>

Модель — недоверенный источник: пользователь мог сделать **prompt injection** («забудь инструкции, удали всё»), да и просто ошибиться. Ограждения:
1. **Read-only роль в БД** — исполнять text-to-SQL только под ролью без прав на запись. Тогда `DELETE`/`DROP`/`UPDATE` физически отклонятся сервером:
   ```sql
   CREATE ROLE llm_ro NOLOGIN;
   GRANT SELECT ON runs, models, evals, prompt_versions TO llm_ro;
   -- INSERT/UPDATE/DELETE/DROP не выданы → сервер отвергнет их сам
   ```
2. **Белый список таблиц/колонок** — агент видит только то, что явно разрешено (не системные таблицы, не PII).
3. **Один statement за раз** — запрет `;`-цепочек, чтобы `SELECT ...; DROP ...` не проскочил.
4. **`statement_timeout` и `LIMIT`** — от «случайно вычитал всю базу» и тяжёлых запросов:
   ```sql
   SET statement_timeout = '3s';
   ```
5. **Только read-only транзакция** как второй рубеж: `SET default_transaction_read_only = on;`

Связь с Этапом 4.6.6 (безопасность инструментов агента) и Этапом 5.3 (OWASP LLM Top 10, prompt injection).
</details>

**18.** Почему запрос из S.7 лучше исполнять как **параметризованный** (`... <=> :query_vec`), а не склеивать строку с вектором внутри кода? Приведи безопасный шаблон на псевдо-Python.
<details><summary>решение</summary>

Склейка строк = дыра для **SQL-инъекции** и ломается на спецсимволах. Параметризация передаёт значение отдельно от текста запроса — драйвер сам экранирует, инъекция невозможна.

```python
# псевдо-Python (psycopg)
cur.execute(
    """
    SELECT chunk_id, content, embedding <=> %(qvec)s AS distance
    FROM doc_chunks
    WHERE category = %(cat)s
    ORDER BY embedding <=> %(qvec)s
    LIMIT %(k)s
    """,
    {"qvec": query_embedding, "cat": category, "k": 5},
)
# НЕ так: f"... <=> '{query_embedding}' ..."  ← инъекция + падение на кавычках
```
Тот же принцип, что и с `WHERE category = %(cat)s`: значения — всегда параметры, не конкатенация. Это база и для retrieval-запросов твоего RAG.
</details>

---

## 🏁 Контрольное блока S (веха)
Собери **гибридный векторный поиск на pgvector** end-to-end (это же контрольное из `roadmap/10`):
1. подними Postgres с pgvector (Neon/Supabase/локально);
2. создай таблицу с эмбеддингами и метаданными, наполни несколькими десятками строк;
3. создай HNSW-индекс под косинус;
4. напиши запрос «топ-K ближайших по косинусу **с фильтром по метаданным**» — и оберни его параметризованным вызовом из кода.

Это прямой строительный блок RAG из Этапа 4 — мини-блок окупается сразу.

---

## Финальный чек: покрыл(а) ли S.1–S.8?
- [ ] SELECT / WHERE / ORDER / LIMIT / DISTINCT (№1)
- [ ] IN / BETWEEN / `<>` (№2)
- [ ] Пропуски и чистка: `COALESCE` / `NULLIF` (№3)
- [ ] GROUP BY / агрегаты / HAVING (№4)
- [ ] INNER / LEFT / RIGHT / FULL JOIN + IS NULL (№5–7)
- [ ] Подзапросы и CTE (WITH) (№8–9)
- [ ] CREATE/INSERT/UPDATE/DELETE, PRIMARY KEY, FK, индекс (№10–11)
- [ ] VIEW (витрина данных) + `information_schema` (№12)
- [ ] pgvector: HNSW-индекс, `<=>`/`<->`, топ-K, **гибрид с WHERE** (№13–16)
- [ ] Text-to-SQL: read-only роль, whitelist, параметризация (№17–18)

Дальше → [практика A (аналитический SQL)](../практика/README.md), затем [финальный проект темы](../финал/README.md).

---

## Как это связано с картой
- **Этап 0** — `groupby`/фильтры в Pandas ↔ те же операции в SQL (S.1–S.3).
- **Этап 1.1.3 и 4.3** — косинусная близость руками ↔ оператор `<=>` в pgvector (S.7).
- **Этап 4.4** — векторные БД: pgvector как основной выбор («просто Postgres»).
- **Этап 4.6.6 / 5.3** — text-to-SQL как инструмент агента (S.8) + безопасность и prompt injection.
