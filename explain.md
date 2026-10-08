# 10 pièges SQL (MySQL) et plus > L'instruction `EXPLAIN`

![images/sql-logo.png](images/sql-logo.png)

> MySQL 8 : sorti en 2018, fin de vie : 2026.

## 1. Présentation de `EXPLAIN`

L'instruction `EXPLAIN` permet d'analyser la manière dont MySQL exécute une requête SQL. Elle est principalement utilisée pour :

* Comprendre le plan d'exécution d'une requête.
* Identifier les problèmes de performances.
* Vérifier l'utilisation des index.
* Optimiser les requêtes lentes.

Plutôt que d'exécuter réellement la requête, MySQL affiche les étapes qu'il prévoit de suivre pour obtenir le résultat.

Syntaxe générale depuis MySQL 8 :

```sql
EXPLAIN FORMAT=TREE
  SELECT *
  FROM customers;
```

> Ancienne syntaxe **avant** MySQL 8 : `EXPLAIN SELECT * FROM customers;`.

## 2. Pourquoi utiliser `EXPLAIN` ?

### 2.a. Détecter les scans complets de tables

Un scan complet signifie que MySQL lit toutes les lignes d'une table.

Exemple :

```sql
EXPLAIN
  SELECT *
  FROM customers
  WHERE lastname = 'Dupont';
```

Si aucun index n'existe sur `lastname`, MySQL devra parcourir toute la table.

### 2.b. Vérifier l'utilisation des index

Un index correctement utilisé améliore considérablement les performances.

```sql
CREATE INDEX idx_customers_lastname
ON customers(lastname);
```

Puis :

```sql
EXPLAIN
  SELECT *
  FROM customers
  WHERE lastname = 'Dupont';
```

L'index devrait apparaître dans le plan d'exécution.

### 2.c. Comprendre l'ordre des jointures

Dans une requête comportant plusieurs tables, MySQL choisit l'ordre des accès.

```sql
EXPLAIN
  SELECT *
  FROM orders o
  INNER JOIN customers c ON c.id = o.customer_id;
```

Le plan indique quelle table est parcourue en premier.

## 3. Les colonnes principales de `EXPLAIN`

Exemple :

```sql
EXPLAIN
  SELECT *
  FROM customers
  WHERE lastname = 'Dupont';
```

Résultat simplifié :

| id | select_type | table     | type | possible_keys          | key                    | rows | Extra       |
| -- | ----------- | --------- | ---- | ---------------------- | ---------------------- | ---- | ----------- |
| 1  | SIMPLE      | customers | ref  | idx_customers_lastname | idx_customers_lastname | 5    | Using where |

### 3.a. id

Identifie chaque étape du plan d'exécution.

```text
1
2
3
```

Plus la requête est complexe (sous-requêtes, unions), plus il peut y avoir d'identifiants.

### 3.b. select_type

Indique le type de requête.

Valeurs courantes :

| Valeur   | Signification          |
| -------- | ---------------------- |
| SIMPLE   | Requête simple         |
| PRIMARY  | Requête principale     |
| SUBQUERY | Sous-requête           |
| DERIVED  | Sous-requête dans FROM |
| UNION    | Partie d'une UNION     |

Exemple :

```sql
SELECT *
FROM products
WHERE category_id IN (SELECT id FROM categories);
```

La sous-requête apparaîtra comme `SUBQUERY`.

### 3.c. table

Indique la table utilisée.

```text
customers
orders
products
```

### 3.d. type

C'est souvent la colonne la plus importante.

Elle indique comment MySQL accède aux données.

Ordre du meilleur au moins bon :

| Type   | Performance |
| ------ | ----------- |
| system | Excellent   |
| const  | Excellent   |
| eq_ref | Très bon    |
| ref    | Bon         |
| range  | Correct     |
| index  | Moyen       |
| ALL    | Mauvais     |

#### const

Une seule ligne est lue.

```sql
EXPLAIN
  SELECT *
  FROM customers
  WHERE id = 10;
```

Avec une clé primaire, MySQL sait exactement où trouver la ligne.

#### eq_ref

Utilisé fréquemment dans les jointures sur clés primaires ou uniques.

```sql
SELECT *
FROM orders o
INNER JOIN customers c
    ON c.id = o.customer_id;
```

#### ref

Recherche sur un index non unique.

```sql
SELECT *
FROM customers
WHERE lastname = 'Dupont';
```

#### range

Recherche sur une plage de valeurs.

```sql
SELECT *
FROM orders
WHERE order_date BETWEEN '2025-01-01'
                     AND '2025-12-31';
```

#### ALL

Scan complet de table.

```sql
SELECT *
FROM customers;
```

ou

```sql
SELECT *
FROM customers
WHERE UPPER(lastname) = 'DUPONT';
```

L'index devient inutilisable.

### 3.e. possible_keys

Liste les index que MySQL pourrait utiliser.

```text
PRIMARY
idx_lastname
idx_city
```

### 3.f. key

Index réellement choisi.

```text
idx_lastname
```

Si cette colonne est vide :

```text
NULL
```

aucun index n'est utilisé.

### 3.g. key_len

Longueur de l'index utilisée.

Exemple :

```text
4
255
768
```

Cette information est utile pour diagnostiquer les index composés.

### 3.h. rows

Nombre estimé de lignes à lire.

```text
5
100
25000
```

Plus cette valeur est faible, mieux c'est généralement.

### 3.i. filtered

Pourcentage estimé de lignes conservées après filtrage.

```text
10.00
50.00
100.00
```

Disponible selon le format utilisé.

### 3.j. Extra

Informations complémentaires.

## 4. Les valeurs importantes de la colonne Extra

### 4.a. Using where

MySQL applique un filtre.

```text
Using where
```

Généralement normal.

### 4.b. Using index

L'index contient toutes les colonnes nécessaires.

```text
Using index
```

On parle alors de **requête couverte (covering index)**.

Exemple :

```sql
SELECT lastname
FROM customers
WHERE lastname = 'Dupont';
```

### 4.c. Using temporary

Création d'une table temporaire.

```text
Using temporary
```

Souvent observé avec :

```sql
GROUP BY
DISTINCT
ORDER BY
```

sur de gros volumes.

### 4.d. Using filesort

MySQL effectue un tri supplémentaire.

```text
Using filesort
```

Exemple :

```sql
SELECT *
FROM customers
ORDER BY lastname;
```

sans index sur `lastname`.

### 4.e. Using join buffer

MySQL doit utiliser une mémoire intermédiaire pour réaliser la jointure.

```text
Using join buffer
```

Souvent signe qu'un index manque sur la clé de jointure.

## 5. `EXPLAIN FORMAT=TREE`

Disponible depuis MySQL 8.

```sql
EXPLAIN FORMAT=TREE
  SELECT *
  FROM customers
  WHERE lastname = 'Dupont';
```

Exemple simplifié :

```text
-> Index lookup on customers
```

Ce format est généralement plus lisible pour comprendre l'ordre réel d'exécution.

## 6. `EXPLAIN ANALYZE`

### 6.a. Principe

Contrairement à `EXPLAIN`, cette commande exécute réellement la requête puis affiche les statistiques mesurées.

```sql
EXPLAIN ANALYZE
  SELECT *
  FROM customers
  WHERE lastname = 'Dupont';
```

### 6.b. Avantages

* Temps réels mesurés.
* Nombre réel de lignes lues.
* Comparaison entre estimation et réalité.
* Diagnostic plus précis.

### 6.c. Inconvénients

La requête est réellement exécutée.

À éviter sur :

* les requêtes très lourdes ;
* les requêtes de modification ;
* les environnements de production sensibles.

## 7. Optimiser une requête à partir de `EXPLAIN`

### 7.a. Ajouter un index

Avant :

```text
type = ALL
rows = 500000
```

Après :

```sql
CREATE INDEX idx_customers_lastname ON customers(lastname);
```

Résultat :

```text
type = ref
rows = 5
```

### 7.b. Éviter les fonctions sur les colonnes indexées

❌ Mauvais :

```sql
SELECT *
FROM customers
WHERE YEAR(created_at) = 2025;
```

✅ Bon :

```sql
SELECT *
FROM customers
WHERE created_at >= '2025-01-01'
  AND created_at < '2026-01-01';
```

### 7.c. Utiliser des index composés

❌ Mauvais :

```sql
WHERE lastname = 'Dupont'
AND city = 'Paris'
```

sans index adapté.

✅ Meilleur :

```sql
CREATE INDEX idx_customer_lastname_city
ON customers(lastname, city);
```

### 7.d. Réduire le nombre de colonnes retournées

❌ Mauvais :

```sql
SELECT *
FROM customers;
```

✅ Meilleur :

```sql
SELECT lastname, firstname
FROM customers;
```

## ✅  8. Bonnes pratiques

* Utiliser `EXPLAIN` avant toute optimisation.
* Vérifier systématiquement la colonne `type`.
* Surveiller les valeurs `ALL`.
* Contrôler la colonne `rows`.
* Rechercher les mentions `Using temporary` et `Using filesort`.
* Vérifier que les index attendus apparaissent dans `key`.
* Tester les requêtes importantes avec `EXPLAIN ANALYZE`.
* Mettre à jour les statistiques avec :

```sql
ANALYZE TABLE customers;
```

lorsque les estimations semblent incohérentes.

## ❌ 9. Limites et pièges

### 9.a. Ce n'est qu'une estimation

`EXPLAIN` affiche souvent des estimations et non des valeurs réelles.

Les chiffres de `rows` peuvent être éloignés de la réalité.

### 9.b. Le meilleur plan peut varier

Après une montée en volume ou une modification des données, MySQL peut choisir un autre index.

Une requête rapide aujourd'hui peut devenir lente plusieurs mois plus tard.

### 9.c. Les statistiques peuvent être obsolètes

Après de nombreuses insertions ou suppressions :

```sql
ANALYZE TABLE ma_table;
```

peut améliorer la qualité du plan d'exécution.

### 9.d. Un index n'est pas toujours utilisé

Même si un index existe, MySQL peut décider qu'un scan complet est plus rapide.

Exemple :

```sql
WHERE status = 'A'
```

si 95 % des lignes possèdent cette valeur.

## 10. Quand utiliser `EXPLAIN` ?

Utilisez `EXPLAIN` lorsque :

* une requête devient lente
* vous créez un nouvel index
* vous développez une requête complexe
* vous utilisez plusieurs jointures
* vous travaillez sur de gros volumes de données
* vous souhaitez valider une optimisation.

## 11. Résumé

| Commande | Résumé en une phrase |
| -------- | -------------------- |
| `EXPLAIN` | Affiche le plan d'exécution estimé par MySQL : ordre des tables, index utilisés, nombre de lignes estimé, type d'accès, etc. |
| `EXPLAIN FORMAT=TREE` | Affiche le plan d’exécution estimé sous forme d'arbre, ce qui permet de visualiser plus facilement l'enchaînement et l'imbrication des opérations. |
| `EXPLAIN ANALYZE` | Exécute réellement la requête et affiche le plan avec les mesures réelles d’exécution (temps, lignes réellement traitées, nombre de boucles), permettant de comparer estimations et réalité. |

L'essentiel à retenir :

* `EXPLAIN` et `EXPLAIN FORMAT=TREE` estiment
* `EXPLAIN ANALYZE` exécute et mesure

Pour l'analyse de performances MySQL, `EXPLAIN` est l'un des outils les plus importants à maîtriser. La combinaison `EXPLAIN`, `EXPLAIN FORMAT=TREE` et `EXPLAIN ANALYZE` permet aujourd'hui de comprendre avec précision comment MySQL exécute une requête et où se situent les éventuels goulots d'étranglement.
