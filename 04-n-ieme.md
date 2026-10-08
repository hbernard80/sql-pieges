# 10 pièges SQL (MySQL) et plus > 4 - Trouver le n-ième

![images/sql-logo.png](images/sql-logo.png)

## 1. Requête

Ecrire une requête pour trouver le n-ième élément, par exemple le deuxième salaire le plus élevé, le troisième produit le plus cher, le dixième meilleur vendeur.

| id | firstname | salary |
| --: | -------- | -----: |
| 1 | Aminata | 33000 |
| 2 | Sébastien | 25000 |
| 3 | Pauline | 27000 |
| 4 | Rachid | 42000 |
| 5 | Gaspard | 25000 |

## 2. Problèmes et solutions

Classons d'abord les salaires par ordre descendant :

```sql
SELECT id, firstname, salary 
FROM employees
ORDER BY salary DESC;
```

| id | firstname | salary |
| --: | -------- | -----: |
| 4 | Rachid | 42000 |
| 1 | Aminata | 33000 |
| 2 | Sébastien | 33000 |
| 3 | Pauline | 27000 |
| 5 | Gaspard | 25000 |

On remarque que le second salaire le plus élevé est 33000 et que cela concerne deux employés (Aminata, Sébastien).

3 solutions :

- avec une sous-requête
- sans sous-requête
- utilisation du fenêtrage

### 2.a. Solution sans sous-requête

On passe le premier résultat, salaire le plus élevé, grâce à OFFSET, mais il faut aussi utiliser `DISTINCT` pour distinguer les ex-aequo (que ce soit pour sauter le premier ou obtenir le second) :

```sql
SELECT id, firstname, DISTINCT(salary) 
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

> En MySql, le terme `OFFSET` peut être omis. On peut donc écrire `LIMIT 1, 1` pour la dernière ligne. Cette syntaxe peut paraître moins explicite.

<!-- -------------------------------------------------------------------------------- -->
<!-- CHATGPT BRUT A REFORMULER 
<!-- /!\ LES VALEURS ET NOMS DE COLONNES NE SONT PAS CELLES DE L'EXEMPLE -->
<!-- -------------------------------------------------------------------------------- -->

## 1. `MAX()` avec une sous-requête

C'est une solution SQL classique :

```sql
SELECT MAX(salary)
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
);
```

La sous-requête :

```sql
SELECT MAX(salary)
FROM employees
```

donne `90000`.

La requête principale cherche ensuite le maximum strictement inférieur à `90000` :

```text
75000
```

C'est probablement **la solution avec sous-requête la plus simple à expliquer en entretien**.

## 2. Sous-requête avec `DISTINCT`, `ORDER BY` et `LIMIT`

On peut d'abord récupérer les deux salaires les plus élevés :

```sql
SELECT MIN(salary)
FROM (
    SELECT DISTINCT salary
    FROM employees
    ORDER BY salary DESC
    LIMIT 2
) AS top_salaries;
```

La sous-requête produit :

| salary |
|---:|
| 90000 |
| 75000 |

Puis :

```sql
MIN(salary)
```

donne :

```text
75000
```

Cette technique peut facilement être adaptée au **Nième salaire** :

```sql
SELECT MIN(salary)
FROM (
    SELECT DISTINCT salary
    FROM employees
    ORDER BY salary DESC
    LIMIT 3
) AS top_salaries;
```

Ici, on obtiendrait le troisième salaire le plus élevé : `65000`.

## 3. Sous-requête corrélée avec `COUNT()`

Une solution plus algorithmique consiste à compter le nombre de salaires distincts supérieurs :

```sql
SELECT DISTINCT e1.salary
FROM employees e1
WHERE 1 = (
    SELECT COUNT(DISTINCT e2.salary)
    FROM employees e2
    WHERE e2.salary > e1.salary
);
```

Pour `75000`, il existe exactement **un salaire distinct supérieur** :

```text
90000
```

Donc `75000` est bien le deuxième salaire distinct le plus élevé.

Cette solution se généralise très bien. Pour le 3e :

```sql
WHERE 2 = (
    SELECT COUNT(DISTINCT e2.salary)
    FROM employees e2
    WHERE e2.salary > e1.salary
);
```

## 4. `DENSE_RANK()` avec une sous-requête

Avec MySQL 8+, les fonctions de fenêtrage donnent une solution très expressive :

```sql
SELECT DISTINCT salary
FROM (
    SELECT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS ranking
    FROM employees
) AS ranked
WHERE ranking = 2;
```

On obtient conceptuellement :

| salary | ranking |
|---:|---:|
| 90000 | 1 |
| 90000 | 1 |
| 75000 | 2 |
| 65000 | 3 |
| 50000 | 4 |

Puis :

```sql
WHERE ranking = 2
```

donne `75000`.

Le choix de **`DENSE_RANK()` plutôt que `ROW_NUMBER()`** est important puisque les deux employés à `90000` doivent partager le même rang.

## 5. `DENSE_RANK()` avec une CTE

Même principe, mais souvent plus lisible :

```sql
WITH ranked_salaries AS (
    SELECT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS ranking
    FROM employees
)
SELECT DISTINCT salary
FROM ranked_salaries
WHERE ranking = 2;
```

Pour moi, c'est la forme la plus claire lorsqu'on travaille avec du **SQL moderne** et que la requête doit évoluer.

## 6. Avec `NOT EXISTS`

On peut aussi raisonner en termes d'existence.

```sql
SELECT DISTINCT e1.salary
FROM employees e1
WHERE EXISTS (
    SELECT 1
    FROM employees e2
    WHERE e2.salary > e1.salary
)
AND NOT EXISTS (
    SELECT 1
    FROM employees e2
    WHERE e2.salary > e1.salary
      AND EXISTS (
          SELECT 1
          FROM employees e3
          WHERE e3.salary > e2.salary
      )
);
```

Cette solution est nettement plus compliquée et n'a pas grand intérêt pratique ici, mais elle montre qu'on peut résoudre le problème uniquement avec des tests d'existence.

## 7. Synthèse

| Solution | Sous-requête | MySQL | Intérêt |
|---|:---:|:---:|---|
| `DISTINCT + ORDER BY + LIMIT/OFFSET` | ❌ | Toutes versions | Très simple |
| `MAX()` + sous-requête | ✅ | Toutes versions | **Simple et élégante pour le 2e** |
| `MIN()` sur les N premiers | ✅ | Toutes versions | Facile à généraliser |
| Sous-requête corrélée + `COUNT` | ✅ | Toutes versions | Généralisation au Nième |
| `DENSE_RANK()` | ✅ | 8+ | **Très bonne solution moderne** |
| CTE + `DENSE_RANK()` | CTE | 8+ | Très lisible |
| Auto-jointure + `MAX()` | ❌ | Toutes versions | Astucieuse sans sous-requête |

Pour un **entretien SQL**, je connaîtrais particulièrement ces trois formes :

```sql
-- Solution classique
SELECT MAX(salary)
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

```sql
-- Solution simple avec LIMIT
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

```sql
-- Solution SQL moderne
WITH ranked AS (
    SELECT salary,
           DENSE_RANK() OVER (ORDER BY salary DESC) AS ranking
    FROM employees
)
SELECT DISTINCT salary
FROM ranked
WHERE ranking = 2;
```

Elles sont aussi intéressantes parce qu'elles permettent d'aborder en entretien la différence entre **`ROW_NUMBER()`, `RANK()` et `DENSE_RANK()`**, qui est une question SQL assez classique.