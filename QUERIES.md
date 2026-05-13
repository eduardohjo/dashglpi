# DashGLPI SQL Queries

Este documento contém as queries SQL reais utilizadas pelo DashGLPI para gerar o Ranking de Técnicos e identificar o Técnico do Mês.

> **Nota:** As queries abaixo foram convertidas do formato DBAL do GLPI para SQL nativo para fins de documentação. Substitua `glpi_` pelo prefixo real do seu banco de dados, se aplicável.

## 1. Ranking de Técnicos (Mensal)

Esta query retorna a lista de técnicos ordenada pela pontuação acumulada no mês atual.

```sql
SELECT
    u.id,
    TRIM(CONCAT(IFNULL(u.firstname, u.name), ' ', IFNULL(u.realname, ''))) AS name,
    UPPER(CONCAT(LEFT(IFNULL(u.firstname, u.name), 1), LEFT(IFNULL(u.realname, ''), 1))) AS avatar,
    COUNT(t.id) AS tickets,
    SUM(CASE WHEN t.time_to_resolve IS NOT NULL AND t.time_to_resolve >= t.solvedate THEN 1 ELSE 0 END) AS sla_ok,
    SUM(CASE WHEN t.time_to_resolve IS NOT NULL AND t.time_to_resolve >= t.solvedate THEN 15 ELSE 10 END) AS points
FROM glpi_tickets AS t
INNER JOIN glpi_tickets_users AS tu ON (tu.tickets_id = t.id)
INNER JOIN glpi_users AS u ON (u.id = tu.users_id)
WHERE
    tu.type = 2                   -- Tipo 'Atribuído a'
    AND t.status IN (5, 6)         -- Solucionado ou Fechado
    AND t.is_deleted = 0
    AND t.solvedate IS NOT NULL
    AND t.solvedate >= DATE_FORMAT(NOW(), '%Y-%m-01') -- Apenas mês atual
    -- Filtro de Entidade (exemplo para entidade 0)
    -- AND t.entities_id = 0
GROUP BY u.id, u.firstname, u.name, u.realname
ORDER BY points DESC
LIMIT 20;
```

### Regras de Pontuação:
- **Chamado Resolvido:** +10 pontos.
- **SLA no Prazo:** +5 pontos bônus (Total 15).

---

## 2. Técnico do Mês

O "Técnico do Mês" é simplesmente o primeiro colocado da query acima. No DashGLPI, ele é extraído programaticamente do topo do ranking.

```sql
SELECT
    TRIM(CONCAT(IFNULL(u.firstname, u.name), ' ', IFNULL(u.realname, ''))) AS name,
    SUM(CASE WHEN t.time_to_resolve IS NOT NULL AND t.time_to_resolve >= t.solvedate THEN 15 ELSE 10 END) AS points
FROM glpi_tickets AS t
INNER JOIN glpi_tickets_users AS tu ON (tu.tickets_id = t.id)
INNER JOIN glpi_users AS u ON (u.id = tu.users_id)
WHERE
    tu.type = 2
    AND t.status IN (5, 6)
    AND t.is_deleted = 0
    AND t.solvedate >= DATE_FORMAT(NOW(), '%Y-%m-01')
GROUP BY u.id
ORDER BY points DESC
LIMIT 1;
```
