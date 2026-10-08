# 10 pièges SQL (MySQL) et plus > 8 - Les sous-requêtes

![images/sql-logo.png](images/sql-logo.png)

En MySQL, les sous-requêtes peuvent poser des problèmes de performances, de résultats incorrects ou inattendus et de gestion des valeurs `NULL`.

Voici les principaux pièges à connaître, avec des contre-exemples, des explications et des solutions adaptées à MySQL 8.x.

## 1. Utiliser `NOT IN` avec des valeurs `NULL`

C'est l'un des pièges les plus classiques.

Contre-exemple :

```sql
SELECT *
FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
```

Problème : si la sous-requête retourne au moins une valeur `NULL`, la condition `NOT IN` peut ne retourner aucune ligne.

Par exemple, si la sous-requête retourne `(1, 2, NULL)`, l'expression `3 NOT IN (1, 2, NULL)` produit `UNKNOWN`, et non `TRUE`.

Solution : utiliser `NOT EXISTS`.

```sql
SELECT *
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

`NOT EXISTS` ne souffre pas de ce problème.

## 2. Utiliser `=` avec une sous-requête retournant plusieurs lignes

Contre-exemple :

```sql
SELECT *
FROM products
WHERE category_id = (SELECT id FROM categories WHERE active = 1);
```

Problème : si plusieurs catégories sont actives, MySQL retourne l'erreur : `ERROR 1242 (21000): Subquery returns more than 1 row`. Solution : utiliser `IN` lorsque plusieurs résultats sont possibles.

```sql
SELECT *
FROM products
WHERE category_id IN (SELECT id FROM categories WHERE active = 1);
```

Une sous-requête utilisée avec `=` doit retourner au maximum une ligne.

## 3. Utiliser des sous-requêtes corrélées coûteuses

Une sous-requête corrélée dépend des valeurs de la requête principale.

Contre-exemple :

```sql
SELECT c.id, 
       c.name,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) AS total_orders
FROM customers c;
```

Problème : la sous-requête peut être exécutée pour chaque client. Avec 100 000 clients, cela peut représenter un travail important. MySQL peut optimiser certaines sous-requêtes corrélées, mais pas systématiquement. Solution : envisager une jointure avec agrégation.

```sql
SELECT c.id, c.name, COUNT(o.id) AS total_orders
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name;
```

Cette approche peut être plus efficace, mais ce n'est pas systématique. Un index sur `orders(customer_id)` est également important.

## 4. Utiliser `IN` au lieu de `EXISTS` pour vérifier une existence

Exemple :

```sql
SELECT *
FROM customers
WHERE id IN (SELECT customer_id FROM orders);
```

Cette requête est correcte, mais lorsqu'on cherche uniquement à savoir si un client possède au moins une commande, `EXISTS` exprime plus directement l'intention. Alternative :

```sql
SELECT *
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

Avantage : `EXISTS` vérifie seulement l'existence d'une ligne correspondante. Attention toutefois : MySQL 8 peut transformer les deux formes en semi-jointures (semijoins). `EXISTS` n'est donc pas systématiquement plus rapide que `IN`.

## 5. Oublier que `LIMIT 1` sans `ORDER BY` est non déterministe

Contre-exemple :

```sql
SELECT *
FROM products
WHERE price = (SELECT price FROM products LIMIT 1);
```

Problème : sans `ORDER BY`, rien ne garantit quelle ligne sera choisie. Solution :

```sql
SELECT *
FROM products
WHERE price = (SELECT price FROM products ORDER BY created_at DESC, id DESC LIMIT 1);
```

Le tri sur `id` permet de départager les produits ayant la même date de création, à condition que `id` soit unique.

## 6. Utiliser inutilement `DISTINCT` dans une sous-requête `EXISTS`

Contre-exemple :

```sql
SELECT *
FROM customers c
WHERE EXISTS (SELECT DISTINCT customer_id FROM orders o WHERE o.customer_id = c.id);
```

Problème : `EXISTS` ne s'intéresse qu'à l'existence d'une ligne, pas à son unicité.

Solution :

```sql
SELECT *
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

Le `DISTINCT` est inutile. MySQL peut l'ignorer dans ce contexte, donc le retirer améliore surtout la lisibilité.

## 7. Utiliser une sous-requête dans `FROM` sans anticiper sa matérialisation

Exemple :

```sql
SELECT *
FROM (SELECT customer_id, SUM(total) AS total_amount
      FROM orders
      GROUP BY customer_id) AS totals
WHERE total_amount > 1000;
```

Problème : MySQL peut matérialiser la sous-requête dans une table temporaire.

Cela peut être coûteux si la table `orders` contient des millions de lignes.

Solution possible : filtrer les groupes avec `HAVING`.

```sql
SELECT customer_id, SUM(total) AS total_amount
FROM orders
GROUP BY customer_id
HAVING SUM(total) > 1000;
```

La seconde version évite la table dérivée explicite. Cependant, MySQL dispose d'optimisations comme derived condition pushdown et peut parfois fusionner les tables dérivées. Il faut donc comparer les plans d'exécution.

## 8. Confondre `WHERE` et `HAVING` dans les sous-requêtes

Contre-exemple :

```sql
SELECT *
FROM customers
WHERE id IN (SELECT customer_id
             FROM orders
             WHERE COUNT(*) > 5
             GROUP BY customer_id);
```

Problème : une fonction d'agrégation comme `COUNT(*)` ne peut pas être utilisée directement dans `WHERE`.

Solution :

```sql
SELECT *
FROM customers
WHERE id IN (SELECT customer_id
             FROM orders
             GROUP BY customer_id
             HAVING COUNT(*) > 5);
```

`WHERE` filtre les lignes avant l'agrégation, tandis que `HAVING` filtre les groupes après l'agrégation.

## 9. Oublier les cas particuliers des sous-requêtes scalaires

Une sous-requête scalaire doit retourner une seule colonne et au maximum une ligne.

Exemple :

```sql
SELECT *
FROM products
WHERE price > (SELECT AVG(price)
               FROM products
               WHERE category_id = 10);
```

Cette requête est valide.

Mais attention :

- Si la sous-requête retourne plusieurs lignes, MySQL déclenche une erreur.
- Si elle retourne plusieurs colonnes, MySQL déclenche également une erreur.
- Si elle ne retourne aucune ligne, sa valeur est `NULL`.
- Si elle retourne `NULL`, les comparaisons classiques (`=`, `>`, `<`) produisent `UNKNOWN`.

Dans ce dernier cas, les lignes concernées sont exclues par `WHERE`.

## 10. Utiliser une sous-requête sans index adapté

Exemple :

```sql
SELECT *
FROM customers c
WHERE EXISTS (SELECT 1
              FROM orders o
              WHERE o.customer_id = c.id
              AND o.status = 'paid');
```

Problème : sans index adapté, MySQL peut devoir examiner un grand nombre de lignes de `orders`.

Solution : créer un index composite.

```sql
CREATE INDEX idx_orders_customer_status ON orders (customer_id, status);
```

Cet index peut accélérer la recherche des commandes payées d'un client.

L'ordre des colonnes de l'index doit être choisi en fonction des requêtes réellement exécutées.

## 11. Utiliser `LIMIT` directement dans une sous-requête avec `IN`

Contre-exemple :

```sql
SELECT *
FROM customers
WHERE id IN (SELECT customer_id
             FROM orders
             ORDER BY created_at DESC
             LIMIT 10);
```

Problème : MySQL 8.x ne prend pas en charge `LIMIT` directement dans certaines sous-requêtes utilisées avec `IN`, `ALL`, `ANY` ou `SOME`.

Il peut retourner l'erreur : `This version of MySQL doesn't yet support 'LIMIT & IN/ALL/ANY/SOME subquery'`

Solution : passer par une table dérivée.

```sql
SELECT *
FROM customers
WHERE id IN (SELECT customer_id
             FROM (SELECT customer_id FROM orders ORDER BY created_at DESC, id DESC LIMIT 10) AS recent_orders
);
```

Attention : cette requête sélectionne les clients associés aux 10 dernières commandes, pas nécessairement 10 clients distincts.

## 12. Tableau récapitulatif

| Piège                     | Risque principal                       | Bonne pratique                           |
| ------------------------- | -------------------------------------- | ---------------------------------------- |
| `NOT IN` avec `NULL`      | Résultats inattendus                   | Préférer `NOT EXISTS`                    |
| `=` avec plusieurs lignes | Erreur SQL                             | Utiliser `IN` si approprié               |
| Sous-requête corrélée     | Coût d'exécution élevé                 | Tester une jointure ou une préagrégation |
| `IN` pour une existence   | Lisibilité / performances potentielles | Envisager `EXISTS`                       |
| `LIMIT` sans tri          | Résultat non déterministe              | Ajouter `ORDER BY`                       |
| `DISTINCT` dans `EXISTS`  | Redondance                             | Supprimer `DISTINCT`                     |
| Sous-requête dans `FROM`  | Matérialisation coûteuse               | Examiner une réécriture                  |
| Agrégation dans `WHERE`   | Erreur SQL                             | Utiliser `HAVING`                        |
| Sous-requête scalaire     | Erreur ou `NULL` inattendu             | Garantir une seule valeur                |
| Index absent              | Scans coûteux                          | Indexer les colonnes utiles              |
| `LIMIT` avec `IN`         | Syntaxe non prise en charge            | Utiliser une table dérivée               |

À retenir : une sous-requête n'est pas intrinsèquement moins performante qu'une jointure. MySQL 8.x sait optimiser de nombreuses sous-requêtes grâce aux semi-jointures, à la matérialisation et à certaines transformations de requêtes. Le bon réflexe est de privilégier d'abord la correction du résultat, puis de comparer les plans d'exécution et les performances sur des volumes représentatifs.
