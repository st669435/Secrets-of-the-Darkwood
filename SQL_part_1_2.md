# Secrets-of-the-Darkwood
Аналитический проект: Исследовательский анализ данных. Решение ad hoc задач от коллег из маркетинговой команды игры «Секреты Темнолесья». Выводы и аналитические комментарии.
* Проект «Секреты Тёмнолесья»
 * Цель проекта: изучить влияние характеристик игроков и их игровых персонажей 
 * на покупку внутриигровой валюты «райские лепестки», а также оценить 
 * активность игроков при совершении внутриигровых покупок
 * 
 * Автор: Клычкова Оксана
 * Дата: 29.12.2025
*/

-- Часть 1. Исследовательский анализ данных
-- Задача 1. Исследование доли платящих игроков 
-- 1.1. Доля платящих пользователей по всем данным:

SELECT 
COUNT(*) AS count_users,-- общее количество игроков
SUM(payer) AS users_pay,-- количество платящих игроков
ROUND(AVG(payer) * 100, 2) AS avg_payer -- доля в процентах
FROM fantasy.users

-- 1.2. Доля платящих пользователей в разрезе расы персонажа:
SELECT race_id AS race,
COUNT(*) AS count_users,-- общее количество игроков
SUM(payer) AS users_pay,-- количество платящих игроков
ROUND(AVG(payer) * 100, 2) AS avg_payer -- доля в процентах
FROM fantasy.users
GROUP BY race_id
ORDER BY race_id;

-- Задача 2. Исследование внутриигровых покупок
-- 2.1. Статистические показатели по полю amount:
SELECT 
COUNT(amount) AS count_amount,--общее количество покупок
SUM(amount) AS sum_amount,--суммарную стоимость всех покупок
MIN(amount) AS min_amount,
MAX(amount) AS max_amount , -- минимальная и максимальная стоимость покупки
AVG(amount) AS avg_amount, -- среднее значение
PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) AS median_amount, -- медиана
STDDEV(amount) AS stddev_amount -- стандартное отклонение стоимости покупки
FROM fantasy.events

-- 2.2: Аномальные нулевые покупки:
SELECT 
COUNT(*) AS total_events,
SUM(CASE WHEN amount = 0 THEN 1 ELSE 0 END) AS zero_amount,
SUM(CASE WHEN amount = 0 THEN 1 ELSE 0 END)::numeric / COUNT(*) * 100 AS part_zero_amount
FROM fantasy.events

-- 2.3: Популярные эпические предметы:
WITH filter_zero AS (
    -- Фильтрация покупок с нулевой стоимостью
SELECT *
FROM fantasy.events
WHERE amount > 0
),
epic_item_stats AS (
SELECT
i.game_items AS item_name,
COUNT(*) AS absolute_sales_count,
COUNT(DISTINCT fs.id) AS unique_buyers_count
FROM filter_zero AS fs
JOIN fantasy.items AS i ON fs.item_code = i.item_code
GROUP BY i.game_items
)
SELECT item_name, absolute_sales_count,
absolute_sales_count * 1.0 / (SELECT COUNT(*) FROM filter_zero) AS relative_sales_share,
unique_buyers_count * 1.0 / (SELECT COUNT(DISTINCT id) FROM filter_zero) AS buyer_share
FROM epic_item_stats
ORDER BY buyer_share DESC;

-- Часть 2. Решение ad hoc-задачи
-- Задача: Зависимость активности игроков от расы персонажа:
-- Общее количество зарегистрированных игроков по расам
WITH registered_players AS (
SELECT race_id,
COUNT(*) as total_registered
FROM fantasy.users
GROUP BY race_id
),
--  количество игроков, которые совершают внутриигровые покупки, и их доля от общего количества зарегистрированных игроков
paying_players AS (
    SELECT 
u.race_id,
COUNT(DISTINCT e.id) as total_paying,
COUNT(DISTINCT e.id)::numeric / COUNT(*) as paying_ratio
FROM fantasy.users u
LEFT JOIN fantasy.events e ON u.id = e.id  AND e.amount > 0  -- Исключаем покупки с нулевой стоимостью
GROUP BY u.race_id
),
--доля платящих игроков среди игроков, которые совершили внутриигровые покупки
player_activity AS (
SELECT u.race_id, e.id,
COUNT(e.id) as purchase_count,
SUM(e.amount) as total_spent,
AVG(e.amount) as avg_purchase_value
FROM fantasy.users AS u
INNER JOIN fantasy.events AS e ON u.id = e.id 
AND e.amount > 0 
GROUP BY u.race_id, e.id
),
-- 1)среднее количество покупок на одного игрока, совершившего внутриигровые покупки 2)средняя стоимость одной покупки на одного игрока, совершившего внутриигровые покупки; 3)средняя суммарная стоимость всех покупок на одного игрока, совершившего внутриигровые покупки.
activity_summary AS (
SELECT race_id,
COUNT(DISTINCT id) as active_paying_players,
AVG(purchase_count) as avg_purchases_per_payer,
AVG(total_spent) as avg_total_spent_per_payer,
AVG(avg_purchase_value) as avg_purchase_value_per_payer
FROM player_activity
GROUP BY race_id
)
SELECT rp.race_id, rp.total_registered,
pp.total_paying as total_paying_players,
-- Доля платящих от общего количества зарегистрированных
pp.paying_ratio  AS paying_ratio_percent,
    -- Доля платящих среди тех, кто вообще покупал
asum.active_paying_players::numeric / pp.total_paying * 100 AS paying_among_buyers_percent,
    -- Среднее количество покупок на одного платящего игрока
asum.avg_purchases_per_payer AS avg_purchases_per_paying_player,
    -- Средняя стоимость одной покупки
asum.avg_purchase_value_per_payer AS avg_purchase_value,
    -- Средняя суммарная стоимость всех покупок на одного платящего игрока
asum.avg_total_spent_per_payer AS avg_total_spent_per_paying_player
FROM registered_players AS rp
LEFT JOIN paying_players AS pp ON rp.race_id = pp.race_id
LEFT JOIN activity_summary AS asum ON rp.race_id = asum.race_id
ORDER BY rp.race_id;
