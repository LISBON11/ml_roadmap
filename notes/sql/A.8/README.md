# A.8 — Ad-hoc методология (декомпозиция через CTE)

> Дистиллят заметок Практикума (практика ad hoc задач).

## Суть простыми словами
**Ad hoc** («под случай») — разовый запрос под конкретный внезапный вопрос бизнеса. Приём: разбить размытую формулировку на части и собрать из них SQL.

## Ключевое
- Приём декомпозиции: «пользователи, заказавшие в **каждом** канале» → `GROUP BY user_id HAVING COUNT(DISTINCT channel) = (SELECT COUNT(DISTINCT channel) ...)`.
- «С нескольких устройств» → `HAVING COUNT(DISTINCT device) > 1`.
- «Топ-1 по метрике» → `ORDER BY <агрегат> DESC LIMIT 1`.
- `CURRENT_DATE` — текущая дата; `CURRENT_DATE - MAX(created_at)` — дней с последнего события.
- `COUNT(DISTINCT ...)` в `HAVING` — рабочая лошадка для условий «во всех / более чем в N».

## Пример
```sql
-- пользователи, заказавшие во ВСЕХ рекламных каналах
SELECT o.user_id
FROM orders o
JOIN sessions s ON o.user_id = s.user_id
GROUP BY o.user_id
HAVING COUNT(DISTINCT s.channel) = (SELECT COUNT(DISTINCT channel) FROM sessions);

-- категория товаров с наибольшим числом уникальных покупателей
SELECT i.category
FROM order_x_item oxi
JOIN items  i ON oxi.item_id = i.item_id
JOIN orders o ON oxi.invoice_id = o.order_id
GROUP BY i.category
ORDER BY COUNT(DISTINCT o.user_id) DESC
LIMIT 1;
```

## Грабли
- «В каждом канале» ≠ «хотя бы в одном»: нужен `= (SELECT COUNT(DISTINCT ...))`, а не `> 0`.
- Забыть `DISTINCT` в `COUNT` при подсчёте уникальных сущностей.

## Докажи себе
- [ ] Разложил(а) устную формулировку задачи на части и собрал(а) запрос через `HAVING COUNT(DISTINCT ...)`.

## Ссылки
-
