# 10 pièges SQL (MySQL) et plus > 7 - Jointures

![images/sql-logo.png](images/sql-logo.png)

Les jointures SQL comportent plusieurs pièges importants, certains liés à la justesse du résultat, d'autres aux performances.

## 1. Oublier la condition de jointure

C'est le piège le plus brutal :

```sql
SELECT *
FROM customers c
JOIN orders o;
```

ou une condition qui ne relie accidentellement pas les deux tables. On obtient alors un produit cartésien : chaque ligne de `customers` est combinée avec chaque ligne de `orders`. Avec 10 000 clients et 100 000 commandes, cela peut représenter :

```text
10 000 × 100 000 = 1 000 000 000 lignes
```

Il faut normalement avoir une condition explicite :

```sql
SELECT *
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

## 2. Faire une jointure sur des colonnes non indexées

Considérons :

```sql
SELECT *
FROM orders o
JOIN customers c ON c.id = o.customer_id;
```

Sur de gros volumes, il est important que les colonnes utilisées pour retrouver les lignes soient correctement indexées.

`customers.id` l'est normalement s'il s'agit de la clé primaire. Il est souvent pertinent d'avoir également :

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

⚠️ Il ne faut pas conclure que toutes les colonnes apparaissant dans un `JOIN` doivent systématiquement recevoir un index. Cela dépend du sens de la jointure, de la cardinalité, des volumes et du plan choisi par l'optimiseur.

> [L'instruction `EXPLAIN`](explain.md) permet de vérifier ce que fait réellement MySQL.

## 3. Utiliser des colonnes de types différents

Par exemple :

```text
customers.id          INT
orders.customer_id    VARCHAR(20)
```

puis :

```sql
ON customers.id = orders.customer_id
```

est une mauvaise conception.

Les conversions implicites peuvent compliquer l'exploitation des index et produire des comportements inattendus.

Les colonnes liées par une clé étrangère devraient utiliser des types compatibles :

```text
customers.id          BIGINT UNSIGNED
orders.customer_id    BIGINT UNSIGNED
```

Il faut également être attentif aux `CHAR`/`VARCHAR`, collations et jeux de caractères lors des jointures textuelles.

## 4. Appliquer une fonction aux colonnes de jointure

On retrouve ici le problème de **sargabilité** :

```sql
SELECT *
FROM customers c
JOIN orders o
    ON LOWER(o.customer_email) = LOWER(c.email);
```

Même si les colonnes sont indexées, les fonctions peuvent empêcher l'utilisation efficace d'index classiques.

Même problème avec :

```sql
ON DATE(a.created_at) = DATE(b.created_at)
```

ou :

```sql
ON CAST(a.code AS CHAR) = b.code
```

Il vaut mieux, lorsque c'est possible, stocker des données comparables et effectuer directement :

```sql
ON o.customer_id = c.id
```

## 5. Transformer involontairement un `LEFT JOIN` en `INNER JOIN`

C'est un grand classique, et probablement l'un des pièges les plus importants à connaître.

On veut récupérer tous les clients, y compris ceux qui n'ont aucune commande :

```sql
SELECT c.*, o.*
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
```

Mais on ajoute :

```sql
SELECT c.*, o.*
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'paid';
```

Pour un client sans commande, les colonnes de `orders` valent `NULL`. La condition :

```sql
WHERE o.status = 'paid'
```

élimine donc ces lignes.

Le `LEFT JOIN` se comporte alors, pour ce filtre, comme un `INNER JOIN`.

Si l'objectif est de conserver tous les clients et de ne joindre que leurs commandes payées :

```sql
SELECT c.*, o.*
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.id
   AND o.status = 'paid';
```

La position d'une condition dans `ON` ou `WHERE` peut donc modifier le résultat, pas seulement les performances.

## 6. Créer des doublons sans comprendre la cardinalité

Supposons :

```text
customers
---------
1 | Alice

orders
------
10 | customer_id = 1
11 | customer_id = 1
12 | customer_id = 1
```

La requête :

```sql
SELECT c.*
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

retournera la valeur _Alice_ trois fois.

Ce n'est pas un doublon au sens SQL : ce sont trois correspondances différentes.

Une erreur fréquente consiste alors à masquer le problème avec :

```sql
SELECT DISTINCT c.*
```

Cela peut fonctionner, mais `DISTINCT` est parfois utilisé comme un **pansement** alors que la véritable question devrait être :

> Quelle est la cardinalité de ma relation : 1:1, 1:N ou N:N ?

Si l'objectif est simplement de rechercher les clients possédant au moins une commande, `EXISTS` exprime souvent mieux l'intention :

```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

## 7. Explosion du nombre de lignes avec plusieurs relations 1:N

Le problème devient encore plus subtil avec plusieurs jointures.

Un produit possède :

* 5 images ;
* 10 commentaires.

On écrit :

```sql
SELECT *
FROM products p
JOIN images i
    ON i.product_id = p.id
JOIN comments c
    ON c.product_id = p.id;
```

Pour un seul produit, on peut obtenir :

```text
5 × 10 = 50 lignes
```

Ce phénomène peut devenir particulièrement dangereux lorsqu'on effectue ensuite des agrégations :

```sql
SUM(...)
COUNT(...)
```

Les valeurs peuvent être artificiellement multipliées. Il faut parfois agréger séparément les tables avant de les joindre.

## 8. Utiliser `JOIN` alors que l'on veut seulement tester une existence

Si l'objectif est _Récupérer les clients qui ont au moins une commande_, on pourrait écrire :

```sql
SELECT DISTINCT c.*
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

Mais la jointure produit d'abord les correspondances, alors que l'on ne s'intéresse pas aux données de `orders`. `EXISTS` correspond directement au besoin :

```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

C'est également plus clair sémantiquement.

## 9. Le piège de `NOT IN` et des `NULL`

Pour trouver les clients sans commande, on rencontre parfois :

```sql
SELECT *
FROM customers
WHERE id NOT IN (
    SELECT customer_id
    FROM orders
);
```

Si la sous-requête peut retourner un `NULL`, le comportement de `NOT IN` peut devenir surprenant à cause de la logique ternaire SQL. Il est généralement plus robuste d'écrire :

```sql
SELECT *
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

Ou, selon le besoin :

```sql
SELECT c.*
FROM customers c
LEFT JOIN orders o
    ON o.customer_id = c.id
WHERE o.id IS NULL;
```

## 10. Joindre sur des colonnes peu sélectives

Une jointure comme :

```sql
ON customer.country = supplier.country
```

peut générer énormément de correspondances. Si 100 000 clients et 5 000 fournisseurs ont `country = 'FR'`, chaque client français peut être associé à tous les fournisseurs français. Une colonne peut donc être parfaitement indexée tout en restant un mauvais critère de jointure si elle possède une faible cardinalité et génère énormément de correspondances.

## 11. Utiliser `SELECT *` sur plusieurs tables

Avec :

```sql
SELECT *
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

on récupère potentiellement beaucoup de colonnes inutiles et plusieurs colonnes portant le même nom :

```text
customers.id
orders.id
customers.created_at
orders.created_at
...
```

C'est moins lisible et augmente également la quantité de données transférée.

Mieux vaut sélectionner explicitement :

```sql
SELECT
    c.id,
    c.name,
    o.id AS order_id,
    o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

## 12. Croire que l'ordre d'écriture des `JOIN` impose l'ordre d'exécution

Dans une requête telle que :

```sql
SELECT ...
FROM customers c
JOIN orders o ON ...
JOIN order_items oi ON ...
JOIN products p ON ...
```

il ne faut pas imaginer MySQL exécutant nécessairement les jointures exactement dans cet ordre. L'optimiseur détermine un plan d'exécution en fonction notamment des index, statistiques, cardinalités estimées et coûts. Pour les requêtes complexes, il faut donc examiner :

```sql
EXPLAIN
SELECT ...
```

et, sur les versions qui le permettent :

```sql
EXPLAIN ANALYZE
SELECT ...
```

> [L'instruction `EXPLAIN ANALYZE`](explain.md) s'avère intéressante pour confronter les estimations de l'optimiseur aux lignes réellement traitées.

## 13. Les pièges à connaître en priorité

Pour une fiche de révision SQL/MySQL, je considérerais ces cinq points comme essentiels :

| Piège | Conséquence |
| ----- | ----------- |
| Condition de jointure oubliée/incorrecte | Produit cartésien |
| Mauvaise cardinalité comprise | Multiplication des lignes |
| `LEFT JOIN` + filtre dans `WHERE` | Perte involontaire des lignes sans correspondance |
| Fonction/conversion sur les colonnes jointes | Index potentiellement mal exploité |
| Plusieurs relations 1:N | Explosion des lignes et agrégats potentiellement faux |

Le point fondamental est que les problèmes de jointures ne sont pas uniquement des problèmes de performances. Une jointure mal conçue peut retourner un résultat parfaitement valide syntaxiquement mais logiquement faux, ce qui est beaucoup plus dangereux qu'une simple requête lente.
