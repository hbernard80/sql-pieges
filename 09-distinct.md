# 10 pièges SQL (MySQL) et plus > 9 - L'instruction `DISTINCT`

![images/sql-logo.png](images/sql-logo.png)

## 1. Introduction : comprendre `DISTINCT`

En MySQL, le mot-clé `DISTINCT` permet de supprimer les doublons dans le résultat d'une requête `SELECT`.

Cependant, plusieurs pièges peuvent entraîner des résultats inattendus, notamment lorsqu'on utilise plusieurs colonnes, des fonctions d'agrégation ou des valeurs `NULL`. Pour illustrer ces différents cas, nous utiliserons une seule table.

### 1.a. Table d'exemple

Contenu de la table `commandes` :

| id | client | ville | produit | montant |
| -- | ------ | ----- | ------- | ------: |
| 1 | Malika | Paris | Clavier | 50 |
| 2 | Malika | Paris | Souris | 20 |
| 3 | Bob | Lyon  | Clavier | 50 |
| 4 | Bob | Paris | Écran | 200 |
| 5 | Charlie | Lyon  | Clavier | 50 |
| 6 | Malika | Paris | Clavier | 50 |
| 7 | `NULL` | Lille | Souris | 20 |

## 2. Piège n° 1 : `DISTINCT` sur une ou plusieurs colonnes

### 2.a. DISTINCT sur une seule colonne

```sql
SELECT DISTINCT client
FROM commandes;
```

Résultat :

| client |
| ------ |
| Malika |
| Bob |
| Charlie |
| `NULL` |

La requête retourne quatre valeurs distinctes, dont une valeur `NULL`.

Les différentes occurrences de `Malika` et `Bob` ont été regroupées.

### 2.b. DISTINCT sur plusieurs colonnes

```sql
SELECT DISTINCT client, ville
FROM commandes;
```

Résultat :

| client | ville |
| ------ | ----- |
| Malika | Paris |
| Bob | Lyon |
| Bob | Paris |
| Charlie | Lyon |
| `NULL` | Lille |

Piège : `DISTINCT` ne s'applique pas uniquement à la colonne qui suit immédiatement le mot-clé.

Il s'applique à l'ensemble des colonnes sélectionnées.

Ainsi, `Bob / Lyon` et `Bob / Paris` sont deux combinaisons distinctes, même si le client est identique.

### 2.c. Ajouter une colonne peut annuler l'effet de DISTINCT

```sql
SELECT DISTINCT id, client, ville
FROM commandes;
```

Cette requête retourne les sept lignes.

Pourquoi ? Parce que `id` est une clé primaire : chaque valeur est unique. Par conséquent, chaque combinaison `(id, client, ville)` est nécessairement unique. Bonne pratique : ne pas inclure une clé primaire dans un `SELECT DISTINCT` si l'objectif est d'éliminer les doublons sur d'autres colonnes.

## 3. Piège n° 2 : `COUNT(colonne)` vs `COUNT(DISTINCT colonne)`

Ces deux expressions ne comptent pas la même chose.

### 3.a. COUNT(colonne)

```sql
SELECT COUNT(client)
FROM commandes;
```

Résultat : `6`

`COUNT(client)` compte toutes les valeurs non `NULL` de la colonne `client`, y compris les doublons.

### 3.b. COUNT(DISTINCT colonne)

```sql
SELECT COUNT(DISTINCT client)
FROM commandes;
```

Résultat : `3`

Cette fois, MySQL compte uniquement les valeurs distinctes et non `NULL` :

- Malika
- Bob
- Charlie

### 3.c. Comparaison avec COUNT(\*)

```sql
SELECT COUNT(*) AS total_lignes,
       COUNT(client) AS clients_non_null,
       COUNT(DISTINCT client) AS clients_uniques
FROM commandes;
```

| total_lignes | clients_non_null | clients_uniques |
| -----------: | ---------------: | --------------: |
| 7            | 6                | 3               |

À retenir :

- `COUNT(*)` compte toutes les lignes, même celles contenant des valeurs `NULL`.
- `COUNT(colonne)` compte les valeurs non `NULL`, y compris les doublons.
- `COUNT(DISTINCT colonne)` compte les valeurs uniques non `NULL`.

## 4. Piège n° 3 : DISTINCT et les valeurs NULL

Contrairement à `COUNT(DISTINCT ...)`, un simple `SELECT DISTINCT` conserve une valeur `NULL` dans le résultat.

```sql
SELECT DISTINCT client
FROM commandes;
```

Résultat : quatre lignes, dont une ligne `NULL`.

En revanche :

```sql
SELECT COUNT(DISTINCT client)
FROM commandes;
```

Résultat : `3`.

Pourquoi cette différence ?

- `SELECT DISTINCT` considère les valeurs `NULL` comme des doublons entre elles pour éliminer les répétitions.
- `COUNT(DISTINCT colonne)` ignore les valeurs `NULL` avant de compter.

Il faut donc distinguer le nombre de lignes retournées par `SELECT DISTINCT` du résultat de `COUNT(DISTINCT ...)`.

## 5. Piège n° 4 : DISTINCT ne s'applique pas à une colonne isolée du SELECT

Cette syntaxe est incorrecte :

```sql
SELECT client, DISTINCT ville
FROM commandes;
```

En MySQL, `DISTINCT` doit apparaître immédiatement après `SELECT` (ou après un éventuel modificateur `ALL`, qui est une alternative, pas un préfixe).

La syntaxe correcte est :

```sql
SELECT DISTINCT client, ville
FROM commandes;
```

Il n'est pas non plus possible d'écrire :

```sql
SELECT DISTINCT(client, ville)
FROM commandes;
```

Cette syntaxe n'est pas une manière valide d'appliquer `DISTINCT` à deux colonnes dans un `SELECT` MySQL.

Attention : les parenthèses autour d'une seule colonne sont autorisées :

```sql
SELECT DISTINCT(client)
FROM commandes;
```

Mais cela est équivalent à :

```sql
SELECT DISTINCT client
FROM commandes;
```

`DISTINCT` reste un modificateur du `SELECT`, pas une fonction appliquée uniquement à `client`.

## 6. Piège n° 5 : COUNT(DISTINCT ...) sur plusieurs colonnes

MySQL autorise une syntaxe particulière :

```sql
SELECT COUNT(DISTINCT client, ville)
FROM commandes;
```

Résultat : `4`.

Les combinaisons comptabilisées sont :

| client | ville |
| ------ | ----- |
| Malika | Paris |
| Bob | Lyon |
| Bob | Paris |
| Charlie | Lyon |

Attention aux valeurs `NULL` : MySQL exclut du comptage toute combinaison contenant au moins une valeur `NULL`. Ainsi, la combinaison `(NULL, Lille)` n'est pas comptabilisée. Il existe donc une différence entre :

```sql
SELECT DISTINCT client, ville
FROM commandes;
```

qui retourne cinq lignes, et :

```sql
SELECT COUNT(DISTINCT client, ville)
FROM commandes;
```

qui retourne `4`. Pour compter toutes les combinaisons distinctes, y compris celles contenant `NULL`, on peut utiliser une sous-requête :

```sql
SELECT COUNT(*)
FROM (
    SELECT DISTINCT client, ville
    FROM commandes
) AS combinaisons;
```

Résultat : `5`.

## 7. Piège n° 6 : DISTINCT ne remplace pas GROUP BY

`DISTINCT` et `GROUP BY` peuvent parfois produire les mêmes résultats, mais ils répondent à des besoins différents. Ces deux requêtes retournent les mêmes clients :

```sql
SELECT DISTINCT client
FROM commandes;
```

```sql
SELECT client
FROM commandes
GROUP BY client;
```

Cependant, si l'objectif est de compter les commandes par client, `GROUP BY` est plus adapté :

```sql
SELECT client, COUNT(*) AS nombre_commandes
FROM commandes
GROUP BY client;
```

Résultat :

| client | nombre_commandes |
| ------: | ---------------: |
| Malika  | 3 |
| Bob | 2 |
| Charlie | 1 |
| `NULL` | 1 |

Bonne pratique :

- Utiliser `DISTINCT` pour éliminer les doublons.
- Utiliser `GROUP BY` pour regrouper des lignes et effectuer des calculs d'agrégation.

## 8. Piège n° 7 : DISTINCT ne garantit pas l'ordre des résultats

Cette requête :

```sql
SELECT DISTINCT ville
FROM commandes;
```

Retourne les villes distinctes, mais leur ordre n'est pas garanti. Même si les résultats semblent triés, il ne faut pas s'appuyer sur ce comportement. Pour obtenir un résultat déterministe :

```sql
SELECT DISTINCT ville
FROM commandes
ORDER BY ville ASC;
```

Résultat :

| ville |
| ----- |
| Lille |
| Lyon  |
| Paris |

Bonne pratique : toujours utiliser `ORDER BY` lorsqu'un ordre précis est nécessaire.

## 9. Piège n° 8 : DISTINCT peut masquer un problème de jointure

Un développeur peut être tenté d'utiliser `DISTINCT` pour supprimer des doublons produits par une jointure :

```sql
SELECT DISTINCT c1.client
FROM commandes c1
JOIN commandes c2 ON c1.client = c2.client;
```

Cette requête produit les noms des clients non `NULL` ayant au moins une correspondance, mais effectue une autojointure inutile et potentiellement coûteuse. Il suffit ici d'écrire :

```sql
SELECT DISTINCT client
FROM commandes;
```

Plus généralement, si une jointure multiplie les lignes, ajouter `DISTINCT` peut masquer une erreur dans les conditions de jointure ou une mauvaise compréhension des relations entre les tables. Bonne pratique : vérifier les conditions `ON` et la cardinalité des relations avant d'ajouter `DISTINCT`. Si le besoin consiste simplement à vérifier l'existence d'une relation, `EXISTS` peut être préférable à une jointure suivie d'un `DISTINCT`.

## 13. Tableau récapitulatif des principaux pièges

| Piège                         | À retenir                                                  |
| ----------------------------- | ---------------------------------------------------------- |
| `DISTINCT colonne1`           | Supprime les valeurs répétées de la colonne.               |
| `DISTINCT colonne1, colonne2` | Élimine les combinaisons identiques.                       |
| `DISTINCT` avec clé primaire  | Généralement inutile, car chaque ligne reste unique.       |
| `COUNT(colonne)`              | Compte les valeurs non `NULL`, doublons inclus.            |
| `COUNT(DISTINCT colonne)`     | Compte les valeurs uniques non `NULL`.                     |
| `DISTINCT` et `NULL`          | Conserve une seule occurrence `NULL`.                      |
| `COUNT(DISTINCT col1, col2)`  | Ignore les combinaisons contenant `NULL`.                  |
| `DISTINCT` vs `GROUP BY`      | Déduplication vs regroupement et agrégation.               |
| `DISTINCT` sans `ORDER BY`    | Aucun ordre garanti.                                       |

Conclusion : le piège principal de `DISTINCT` est de confondre l'unicité d'une valeur, l'unicité d'une combinaison de colonnes et l'unicité d'une ligne métier. Ces trois notions ne sont pas nécessairement équivalentes.
