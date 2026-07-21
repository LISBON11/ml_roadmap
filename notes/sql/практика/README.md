# Практика A — Аналитический SQL (закрепление спринтов Практикума)

Набор заданий под **блок A**: множества, `CASE`, даты, **оконные функции**, ранжирование, смещения, перцентили, ad-hoc-декомпозиция — всё из твоих спринтов 1–2 Практикума, но на данных, похожих на реальную работу AI Engineer: логи прогонов LLM, метрики eval, версии промптов, обратная связь.

> **Порядок прохождения:**
> 1. Сначала → [практика S (рабочий SQL для LLM)](../S-практика/README.md): выборки, джойны, CTE, изменение данных, **pgvector**, text-to-SQL.
> 2. Затем → **этот файл** (аналитика A.1–A.8).
> 3. В конце → [**финальный проект темы (RAG-аналитика на SQL)**](../финал/README.md) — капстоун на весь S+A сразу.
>
> Базовые `SELECT`/`JOIN`/агрегаты (S.1–S.4) намеренно **не дублируются** здесь — они в S-практике. Тут сразу аналитика.

---

## Песочница

1. **Быстрый прогон — [DB Fiddle](https://www.db-fiddle.com/)** *(проверено на июль 2026).* Выбери **PostgreSQL**, вставь seed ниже в «Schema SQL», запросы — справа, Run.
2. **Постоянная база — [Neon](https://neon.tech/) / [Supabase](https://supabase.com/)** *(бесплатный tier).* Заливаешь seed один раз — база живёт; к ней же подключишь pgvector из S-практики.
3. **Тренажёр с автопроверкой — [PGExercises](https://pgexercises.com/)**: разделы **window functions** и recursive отшлифуют технику A.4–A.6 на нейтральных данных.

**Мой план:** (1) прогони PGExercises до window functions — отточит синтаксис; (2) затем задания ниже на seed-данных — перенесут навык на ML-контекст.

---

## Seed-схема (та же, что в S-практике — скопируй один раз)

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

-- Метрики eval (по прогонам)
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

-- Обратная связь пользователей
CREATE TABLE feedback (
  fb_id      int PRIMARY KEY,
  run_id     int REFERENCES runs,
  user_id    int,
  rating     int,   -- 1..5
  created_at timestamp
);
INSERT INTO feedback VALUES
 (1,1,101,5,'2026-06-21 09:10'),(2,2,102,4,'2026-06-21 09:20'),
 (3,3,101,3,'2026-06-21 10:30'),(4,6,103,5,'2026-06-22 12:30'),
 (5,8,104,2,'2026-06-23 09:30'),(6,9,102,5,'2026-06-24 14:30'),
 (7,11,105,4,'2026-06-25 09:30'),(8,14,101,5,'2026-06-26 10:40');
```
</details>

> 💡 В нескольких заданиях используется **стоимость прогона** — её как CTE `costs` ты собрала в [S-практике №9](../S-практика/README.md): `tokens_in/1000*price_in + tokens_out/1000*price_out`. Здесь она встроена в решения там, где нужна.

---

## Задания (от простого к сложному)

Каждое — реальный вопрос ML-инженера. Реши сам, потом раскрой решение. Цель — покрыть **все функции A.1–A.8**.

### Множества и подзапросы (A.1)
**1.** Список `user_id`, которые оставляли feedback, но НЕ по промпту `classifier` (через `EXCEPT`).
<details><summary>решение</summary>

```sql
SELECT DISTINCT user_id FROM feedback
EXCEPT
SELECT f.user_id FROM feedback f
JOIN runs r ON r.run_id=f.run_id
JOIN prompt_versions p ON p.pv_id=r.pv_id
WHERE p.prompt='classifier';
```
`EXCEPT` = разность множеств: «все, кто оставлял feedback» минус «кто оставлял по classifier».
</details>

**1б.** Собери отчёт «вовлечённые пользователи» из **двух источников** через `UNION` (довольные: `rating >= 4` + все, кто оценивал `support_rag`) и оберни результат в отдельный `CTE`, из которого и выбираешь.
<details><summary>решение</summary>

```sql
WITH engaged AS (
  SELECT user_id FROM feedback WHERE rating >= 4          -- источник 1: довольные
  UNION                                                    -- объединение множеств (дубли убираются)
  SELECT f.user_id FROM feedback f                          -- источник 2: оценившие support_rag
  JOIN runs r ON r.run_id=f.run_id
  JOIN prompt_versions p ON p.pv_id=r.pv_id
  WHERE p.prompt='support_rag'
)
SELECT user_id FROM engaged ORDER BY user_id;
```
`UNION` склеивает две выборки в одно множество и **убирает дубли** (если дубли не важны — `UNION ALL`, он быстрее). Обёртка в `CTE` — тот самый приём «собрал отчёт → фильтруешь/сортируешь его отдельным шагом».
</details>

### CASE и категоризация (A.2)
**2.** Пометь каждый прогон по латентности: `<1000` → 'fast', `1000–2000` → 'ok', иначе 'slow'. Посчитай, сколько прогонов в каждой категории.
<details><summary>решение</summary>

```sql
SELECT CASE WHEN latency_ms<1000 THEN 'fast'
            WHEN latency_ms<=2000 THEN 'ok' ELSE 'slow' END AS bucket,
       COUNT(*)
FROM runs GROUP BY 1 ORDER BY 1;
```
`CASE` разбивает числовую метрику на корзины — прямой аналог `pd.cut` из Этапа 0.
</details>

**2б. Работа с дубликатами.** Представь, что в `feedback` завёлся неявный дубль — один пользователь оценил один прогон дважды. Оставь по **одной** строке на пару `(run_id, user_id)` — самую свежую — через `ROW_NUMBER`.
<details><summary>решение</summary>

```sql
WITH ranked AS (
  SELECT *,
         ROW_NUMBER() OVER (PARTITION BY run_id, user_id ORDER BY created_at DESC) AS rn
  FROM feedback
)
SELECT fb_id, run_id, user_id, rating, created_at
FROM ranked
WHERE rn = 1;   -- одна строка на ключ = дедупликация
```
Канонический паттерн дедупликации: пронумеровать строки внутри группы-ключа и оставить `rn = 1`. На текущем seed дублей нет — но именно так чистят «неявные дубликаты» в реальных данных (тот же приём «top-1 per group», что в №6). Явные полные дубли строк убирает `SELECT DISTINCT` / `GROUP BY` по всем колонкам.
</details>

### Дата и время (A.3)
**3.** Число прогонов и суммарные выходные токены **по дням**, только за последние 5 дней от `2026-06-27`.
<details><summary>решение</summary>

```sql
SELECT date_trunc('day', created_at) AS day, COUNT(*), SUM(tokens_out)
FROM runs
WHERE created_at >= DATE '2026-06-27' - INTERVAL '5 days'
GROUP BY 1 ORDER BY 1;
```
Покрывает: `date_trunc`, `INTERVAL`, группировку по периоду.
</details>

### Оконные функции: агрегаты (A.4)
**4.** Для каждого прогона выведи его латентность И среднюю латентность его модели рядом (без схлопывания строк).
<details><summary>решение</summary>

```sql
SELECT run_id, model_id, latency_ms,
       AVG(latency_ms) OVER (PARTITION BY model_id) AS model_avg
FROM runs;
```
Ключевое отличие от `GROUP BY`: окно **не схлопывает** строки — каждая строка остаётся, а рядом появляется агрегат по её группе.
</details>

**5.** Накопительная сумма выходных токенов по времени (running total).
<details><summary>решение</summary>

```sql
SELECT run_id, created_at, tokens_out,
       SUM(tokens_out) OVER (ORDER BY created_at) AS running_total
FROM runs;
```
`ORDER BY` внутри `OVER` превращает агрегат в нарастающий итог.
</details>

### Ранжирование (A.5)
**6.** Для каждого промпта оставь только **последнюю версию** (`ROW_NUMBER` + `PARTITION BY prompt ORDER BY version DESC`).
<details><summary>решение</summary>

```sql
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY prompt ORDER BY version DESC) AS rn
  FROM prompt_versions)
SELECT prompt, version, created_at FROM ranked WHERE rn=1;
```
Классический паттерн «top-1 per group».
</details>

**7.** Разбей прогоны на 4 корзины по стоимости через `NTILE(4)` и выведи границы (min/max cost) каждой корзины.
<details><summary>решение</summary>

```sql
WITH c AS (
  SELECT r.run_id,
         r.tokens_in/1000.0*m.price_in + r.tokens_out/1000.0*m.price_out AS cost
  FROM runs r JOIN models m ON m.model_id=r.model_id),
b AS (SELECT *, NTILE(4) OVER (ORDER BY cost) AS q FROM c)
SELECT q, MIN(cost), MAX(cost), COUNT(*) FROM b GROUP BY q ORDER BY q;
```
`NTILE(4)` = квартили: делит строки на 4 примерно равные группы по стоимости.
</details>

### Функции смещения (A.6)
**8.** Для каждой модели посчитай, насколько латентность прогона отличается от **предыдущего** прогона той же модели (`LAG`).
<details><summary>решение</summary>

```sql
SELECT run_id, model_id, created_at, latency_ms,
       latency_ms - LAG(latency_ms) OVER (PARTITION BY model_id ORDER BY created_at) AS delta
FROM runs ORDER BY model_id, created_at;
```
`LAG` заглядывает на строку назад внутри окна — так считают прирост к предыдущему периоду.
</details>

**9.** Для каждого промпта покажи первую и последнюю (по времени) оценку faithfulness через `FIRST_VALUE`/`LAST_VALUE` (осторожно с рамкой окна).
<details><summary>решение</summary>

```sql
SELECT DISTINCT p.prompt,
  FIRST_VALUE(e.score) OVER w AS first_score,
  LAST_VALUE(e.score)  OVER w AS last_score
FROM evals e
JOIN runs r ON r.run_id=e.run_id
JOIN prompt_versions p ON p.pv_id=r.pv_id
WHERE e.metric='faithfulness'
WINDOW w AS (PARTITION BY p.prompt ORDER BY r.created_at
             ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING);
```
Грабли: без явной рамки `LAST_VALUE` вернёт текущую строку, а не конец окна.
</details>

### Описательная статистика (A.7) — ML-специфика (eval)
**10.** По каждой метрике eval посчитай среднее, стандартное отклонение и **медиану** (`PERCENTILE_CONT(0.5)`), а также p95 латентности прогонов.
<details><summary>решение</summary>

```sql
-- по метрикам
SELECT metric, AVG(score), STDDEV(score),
       PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY score) AS median
FROM evals GROUP BY metric;

-- p95 латентности
SELECT PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95
FROM runs WHERE status='ok';
```
p95 — то, чем реально меряют латентность в проде (Этап 5.3). Мост к Этапу 1.4: те же меры центра/разброса, что в статистике.
</details>

### Ad-hoc: многошаговая задача (A.8) — главный навык
**11.** Бизнес-вопрос: **«Для промпта `support_rag` — какая версия дала лучшую среднюю faithfulness, и сколько она стоила в среднем за прогон?»** Реши цепочкой CTE (декомпозиция: прогоны support_rag → их стоимость и faithfulness → агрегат по версии → выбор лучшей).
<details><summary>решение</summary>

```sql
WITH sr AS (  -- прогоны support_rag с версией, стоимостью, faithfulness
  SELECT r.run_id, p.version,
         r.tokens_in/1000.0*m.price_in + r.tokens_out/1000.0*m.price_out AS cost,
         e.score AS faith
  FROM runs r
  JOIN prompt_versions p ON p.pv_id=r.pv_id AND p.prompt='support_rag'
  JOIN models m ON m.model_id=r.model_id
  LEFT JOIN evals e ON e.run_id=r.run_id AND e.metric='faithfulness'),
agg AS (  -- агрегат по версии
  SELECT version, AVG(faith) AS avg_faith, AVG(cost) AS avg_cost
  FROM sr GROUP BY version)
SELECT * FROM agg ORDER BY avg_faith DESC NULLS LAST LIMIT 1;
```
Это ровно то, как выглядит анализ eval-логов в работе AI Engineer — навык из Этапа 4.8 / 5.3: разбить вопрос на шаги и собрать ответ цепочкой CTE.
</details>

---

## После A → финальный проект темы
Проверку «на весь SQL сразу» (S+A) вынесли в отдельный мини-проект: **[Финальный проект — RAG-аналитика на SQL](../финал/README.md)** (полный аналитический отчёт по версиям промпта + RAG-retrieval с аналитикой). Пройди его после этой практики — он закрывает тему.

---

## Финальный чек: покрыл(а) ли A.1–A.8?
- [ ] UNION / EXCEPT / подзапросы / CTE (№1, 1б)
- [ ] CASE + работа с дубликатами (ROW_NUMBER) (№2, 2б)
- [ ] Дата/время, INTERVAL, date_trunc (№3)
- [ ] Оконные агрегаты, running total (№4–5)
- [ ] ROW_NUMBER / NTILE (№6–7)
- [ ] LAG/LEAD, FIRST/LAST_VALUE (№8–9)
- [ ] Перцентили / описательная статистика (№10)
- [ ] Ad-hoc через CTE (№11)

Закрыл(а) A.1–A.8 (плюс [S-практику](../S-практика/README.md)) → переходи к [финальному проекту темы](../финал/README.md), он закрывает SQL для роли AI Engineer.

---

## Нужна проверка по конкретной теме?
Возьми мини-промпт из `07_сводка_тесты_промпт.md`, подставь тему (напр. «оконные функции ранжирования в PostgreSQL») — получишь дополнительное задание с критерием.
