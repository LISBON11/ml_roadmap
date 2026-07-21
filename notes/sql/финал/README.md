# Финальный проект темы — RAG-аналитика на SQL (оверол S+A)

Мини-проект, который проверяет **всю тему SQL сразу** — и рабочий SQL из блока S, и аналитику из блока A. Проходишь после обеих практик ([S](../S-практика/README.md) и [A](../практика/README.md)). Если собираешь оба запроса без подсказок — тема SQL для роли закрыта.

Это не новый материал, а **сборка**: те же seed-данные (логи LLM, eval, эмбеддинги), но каждая задача заставляет соединить навыки из разных блоков в один осмысленный запрос — ровно как в реальной работе AI Engineer над eval-логами и retrieval-слоем RAG.

**Данные:** основной seed (`models`/`prompt_versions`/`runs`/`evals`/`feedback`) — из любой из практик; для задачи 2 — `doc_chunks` из [S-практики (раздел pgvector)](../S-практика/README.md#postgresql--pgvector-s7--ядро-блока).

---

## Задача 1 — Полный аналитический отчёт по версиям промпта
*(S.3–S.5 + A.3–A.8, чистый Postgres, без pgvector)*

**Бизнес-вопрос:** как эволюционировали версии промпта `support_rag` — по качеству, стоимости и латентности?

Одним запросом (цепочкой CTE) собери таблицу по версиям промпта `support_rag`:
- номер версии;
- число прогонов;
- средняя стоимость за прогон;
- средняя faithfulness;
- p95 латентности;
- **дельта средней faithfulness к предыдущей версии**;
- **ранг версии по faithfulness**.

Отсортируй по версии.

<details><summary>подсказка к декомпозиции</summary>

CTE-шаги:
1. `base` — `JOIN` runs + prompt_versions + models + evals, только `support_rag`, со стоимостью и метриками (S.4, S.5);
2. `per_version` — `GROUP BY version`: `COUNT`, `AVG(cost)`, `AVG(faith)`, `PERCENTILE_CONT(0.95)` по латентности (S.3, A.7);
3. финальный `SELECT` с оконными `LAG(avg_faith) OVER (ORDER BY version)` для дельты и `RANK() OVER (ORDER BY avg_faith DESC)` для ранга (A.4–A.6).

```sql
WITH base AS (
  SELECT p.version,
         r.tokens_in/1000.0*m.price_in + r.tokens_out/1000.0*m.price_out AS cost,
         r.latency_ms,
         e.score AS faith
  FROM runs r
  JOIN prompt_versions p ON p.pv_id = r.pv_id AND p.prompt = 'support_rag'
  JOIN models m          ON m.model_id = r.model_id
  LEFT JOIN evals e      ON e.run_id = r.run_id AND e.metric = 'faithfulness'
),
per_version AS (
  SELECT version,
         COUNT(*)                                              AS runs,
         ROUND(AVG(cost), 4)                                   AS avg_cost,
         ROUND(AVG(faith), 3)                                  AS avg_faith,
         PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_latency
  FROM base
  GROUP BY version
)
SELECT version, runs, avg_cost, avg_faith, p95_latency,
       ROUND(avg_faith - LAG(avg_faith) OVER (ORDER BY version), 3) AS d_faith,
       RANK() OVER (ORDER BY avg_faith DESC)                        AS faith_rank
FROM per_version
ORDER BY version;
```
В одном запросе собраны: S.4 (JOIN) + S.5 (CTE) + A.3/A.7 (даты/перцентили) + A.4–A.6 (окна) + A.8 (декомпозиция). Это и есть «анализ eval-логов» из Этапа 4.8 / 5.3.
</details>

**Критерий зачёта:** запрос выполняется, отчёт читается как готовая витрина, дельта и ранг посчитаны оконными функциями (а не подзапросами руками).

---

## Задача 2 — RAG-retrieval с аналитикой поверх
*(S.7 + S.8 + A.4, на Neon/Supabase/локально с pgvector)*

**Бизнес-вопрос:** вернуть релевантные чанки для запроса пользователя и оценить «уверенность» выдачи.

Напиши **один** запрос, который для запроса-вектора `[0.85, 0.15, 0.05]`:
- возвращает топ-3 ближайших по косинусу чанка **с фильтром `category = 'faq'`** (гибридный поиск, S.7);
- рядом выводит **долю дистанции каждого чанка от суммы дистанций тройки** — оконная `SUM(...) OVER ()` как грубая нормировка «уверенности» (A.4).

Затем **словами** сформулируй, как обернуть его безопасным параметризованным вызовом под read-only ролью (S.8).

<details><summary>подсказка к решению</summary>

```sql
WITH top AS (
  SELECT chunk_id, content,
         embedding <=> '[0.85, 0.15, 0.05]' AS dist
  FROM doc_chunks
  WHERE category = 'faq'
  ORDER BY embedding <=> '[0.85, 0.15, 0.05]'
  LIMIT 3
)
SELECT chunk_id, content, dist,
       dist / SUM(dist) OVER () AS dist_share
FROM top
ORDER BY dist;
```

Обёртка в коде (псевдо-Python, S.8):
```python
cur.execute(
    """
    WITH top AS (
      SELECT chunk_id, content, embedding <=> %(qvec)s AS dist
      FROM doc_chunks
      WHERE category = %(cat)s
      ORDER BY embedding <=> %(qvec)s
      LIMIT %(k)s
    )
    SELECT chunk_id, content, dist, dist / SUM(dist) OVER () AS dist_share
    FROM top ORDER BY dist
    """,
    {"qvec": query_embedding, "cat": category, "k": 3},
)   # исполняется под ролью с одним GRANT SELECT — никаких DELETE/DROP
```
Здесь встречаются S.7 (гибридный `<=>` + `WHERE`), A.4 (оконная нормировка без схлопывания) и S.8 (параметры + read-only роль). Это буквально ядро retrieval-слоя твоего RAG из Этапа 4 + аналитика поверх него.
</details>

**Критерий зачёта:** один запрос делает и векторный поиск, и фильтр по метаданным, и оконную нормировку; ты можешь объяснить, почему вектор и категория передаются параметрами, а роль — read-only.

---

## Что дальше
Тема SQL закрыта. В **Этапе 4** (RAG, векторные БД, агенты) обе задачи оживают уже как слои приложения; в **Этапе 5** отчёт из задачи 1 превращается в eval-гейт и cost-аналитику. Перепиши тогда анализ eval-логов своего RAG как ad-hoc-запрос — это станет аккуратным пунктом портфолио.

← назад: [Практика S](../S-практика/README.md) · [Практика A](../практика/README.md)
