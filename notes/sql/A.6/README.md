# A.6 — Смещение: LEAD/LAG, FIRST_VALUE/LAST_VALUE

> Дистиллят заметок Практикума (оконные функции смещения).

## Суть простыми словами
Функции, которые внутри окна берут значение из **другой** строки: первое/последнее в окне или соседнее (предыдущее/следующее).

## Ключевое
- `FIRST_VALUE(col)` / `LAST_VALUE(col)` — первое/последнее значение окна.
- `LEAD/LAG` — значение следующей/предыдущей строки (для разниц «день к дню»).
- **Грабля с рамкой (frame):** по умолчанию окно = «от начала до текущей строки». Поэтому `LAST_VALUE()` без расширения рамки вернёт значение **текущей** строки, а не реально последней.
- Чтобы `LAST_VALUE()` брал последнюю строку окна, расширь рамку:
  `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

## Пример
```sql
SELECT user_id, event_dt, revenue,
       LAST_VALUE(event_dt) OVER (
           PARTITION BY user_id ORDER BY event_dt
           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS last_order_dt
FROM orders;
```

## Грабли
- `LAST_VALUE()` без явной рамки почти всегда даёт «не то» (значение текущей строки) — всегда добавляй `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`.

## Докажи себе
- [ ] Получил(а) дату последнего заказа пользователя через `LAST_VALUE()` с расширенной рамкой.

## Ссылки
-
