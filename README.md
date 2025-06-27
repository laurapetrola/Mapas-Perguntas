# Mapas-Questions

# SQL Optimization Heuristics for Mapas Culturais Queries

This document presents optimized (heuristic-based) and non-optimized SQL queries used to answer specific analytical questions over the *Mapas Culturais* dataset. The goal is to illustrate the impact of heuristic-based SQL rewriting strategies on query performance and clarity.

## Heuristics Applied
- Eliminate correlated subqueries (`= (SELECT ...)`) in favor of CTEs or `JOIN`s with `GROUP BY`.
- Replace `> ALL (subquery)` with `> MAX(subquery)`.
- Move arithmetic expressions out of comparisons (`x / 2 = 100` → `x = 200`).
- Avoid unnecessary `GROUP BY` and prefer `DISTINCT` when appropriate.

---

## 1. Quais são os eventos mais recentes de cada agente, com a data e o nome do evento?

###  With heuristics (CTE)
```sql
WITH latest_events AS (
  SELECT
    e.agent_id,
    MAX(eo.starts_on) AS latest_event_date
  FROM
    event e
  JOIN event_occurrence eo ON e.id = eo.event_id
  GROUP BY e.agent_id
)
SELECT
  a.id AS agent_id,
  a.name AS agent_name,
  e.name AS event_name,
  le.latest_event_date
FROM agent a
JOIN latest_events le ON a.id = le.agent_id
JOIN event e ON e.agent_id = a.id
JOIN event_occurrence eo ON e.id = eo.event_id AND eo.starts_on = le.latest_event_date
ORDER BY a.id;
```
⏱️ **Query executed in 0.5973 seconds**

### Without heuristics (correlated subquery)
```sql
SELECT
  a.id AS agent_id,
  a.name AS agent_name,
  e.name AS event_name,
  eo.starts_on AS event_date
FROM agent a
JOIN event e ON e.agent_id = a.id
JOIN event_occurrence eo ON eo.event_id = e.id
WHERE eo.starts_on = (
  SELECT MAX(eo2.starts_on)
  FROM event_occurrence eo2
  JOIN event e2 ON e2.id = eo2.event_id
  WHERE e2.agent_id = a.id
)
ORDER BY eo.starts_on DESC;
```
⏱️ **Query executed in 3.9665 seconds**

---

## 2. Quais agentes estão localizados em Fortaleza?

### With heuristics
```sql
SELECT DISTINCT a.name AS agent_name, u.email AS agent_email
FROM agent a
JOIN usr u ON a.user_id = u.id
JOIN registration r ON r.agent_id = a.id
JOIN opportunity o ON r.opportunity_id = o.id
JOIN agent_meta am ON am.object_id = a.id AND am.key = 'geoMunicipio'
WHERE am.value = 'Fortaleza'
ORDER BY a.name;
```
⏱️ **Query executed in 0.4096 seconds**

### Without heuristics (GROUP BY desnecessário — com erro)
```sql
SELECT
  a.id AS agent_id,
  a.name AS agent_name,
  a.create_timestamp,
  u.email AS user_email,
  COUNT(ar.id) AS total_relations,
  COUNT(f.id) AS total_files
FROM agent a
JOIN usr u ON a.user_id = u.id
LEFT JOIN agent_relation ar ON ar.agent_id = a.id
LEFT JOIN file f ON f.object_id = a.id AND f.object_type = 'MapasCulturais\Entities\Agent'
GROUP BY a.id, a.name, a.create_timestamp, u.email
ORDER BY a.create_timestamp DESC
LIMIT 10;
```
⏱️ **Query executed in 3.1869 seconds**

---

## 3. Quais eventos têm duração maior que todos os eventos da cidade de Recife?

### 🔍 With heuristics (using MAX)
```sql
WITH eventos_recife AS (
  SELECT
    e.id,
    e.name,
    eo.starts_at,
    eo.ends_at,
    (eo.ends_at - eo.starts_at) AS duracao
  FROM event e
  JOIN event_occurrence eo ON e.id = eo.event_id
  JOIN space s ON eo.space_id = s.id
  JOIN space_meta sm ON s.id = sm.object_id
  WHERE sm.key = 'geoMunicipio' AND sm.value = 'Recife'
)
SELECT
  e.id,
  e.name,
  eo.starts_at,
  eo.ends_at,
  (eo.ends_at - eo.starts_at) AS duracao
FROM event e
JOIN event_occurrence eo ON e.id = eo.event_id
WHERE (eo.ends_at - eo.starts_at) > (SELECT MAX(duracao) FROM eventos_recife)
ORDER BY duracao DESC;
```
⏱️ **Query executed in 0.3145 seconds**

### Without heuristics (using  `> ALL`)
```sql
SELECT e.*
FROM event e
JOIN event_occurrence eo ON e.id = eo.event_id
WHERE (eo.ends_at - eo.starts_at) > ALL (
  SELECT (eo2.ends_at - eo2.starts_at)
  FROM event e2
  JOIN event_occurrence eo2 ON e2.id = eo2.event_id
  JOIN space s ON eo2.space_id = s.id
  WHERE s.name LIKE '%Recife%'
);
```
⏱️ **Query executed in 3.7535 seconds**

---

## 4. Quais espaços têm metade da capacidade igual a 100 pessoas?

### With heuristics (moving arithmetic expression to the other side) 
```sql
SELECT s.name
FROM space s
JOIN space_meta sm ON sm.object_id = s.id AND sm.key = 'capacidade'
WHERE CAST(sm.value AS INTEGER) = 200;
```
⏱️ **Query executed in 0.2304 seconds**

### Without heuristics
```sql
SELECT s.name
FROM space s
WHERE EXISTS (
  SELECT *
  FROM space_meta sm_capacity
  WHERE sm_capacity.object_id = s.id
    AND sm_capacity.key = 'capacidade'
    AND CAST(sm_capacity.value AS INTEGER) / 2 = 100
);
```
⏱️ **Query executed in 0.1908 seconds**

---

## 📌 Summary

These examples illustrate how simple SQL heuristics — like avoiding correlated subqueries, rewriting arithmetic conditions, and using aggregation — can significantly improve performance and maintainability in real-world data analytics, especially when working with public datasets such as *Mapas Culturais*.

> Queries labeled *with heuristic* consistently performed faster and are recommended for scalable data analysis.

