# 10 pièges SQL (MySQL) et plus > 5 - Les index

![images/sql-logo.png](images/sql-logo.png)

## 1. Fonctions appliquées sur une colonne indexée

L'application d'une fonction à une colonne dans une clause `WHERE` peut empêcher MySQL d'exploiter efficacement un index classique (type b-tree) sur cette colonne. Prenons une table `users` possédant un index sur `email` :

```sql
CREATE INDEX idx_users_email
ON users (email);
```

On souhaite effectuer une recherche sans tenir compte des majuscules :

```sql
SELECT *
FROM users
WHERE LOWER(email) = 'john@example.com';
```

## 2. Le problème

L'index contient les valeurs de la colonne `email`, et non nécessairement le résultat de l'expression `LOWER(email)`. Conceptuellement, l'index peut contenir :

```text
Alice@example.com
John@example.com
Paul@example.com
...
```

alors que la requête demande à MySQL de comparer :

```text
LOWER('Alice@example.com')
LOWER('John@example.com')
LOWER('Paul@example.com')
...
```

Avec un index classique sur `email`, le moteur ne peut pas nécessairement rechercher directement :

```text
john@example.com
```

dans les valeurs résultant de `LOWER(email)`. Il peut donc être amené à calculer la fonction pour un grand nombre de lignes, ce qui réduit fortement, voire empêche, l'intérêt de l'index. La condition est alors dite non sargable ou moins sargable.

## 3. Contre-exemple

```sql
SELECT id, email, name
FROM users
WHERE LOWER(email) = 'john@example.com';
```

Sur une table contenant plusieurs millions d'utilisateurs, cette écriture peut être nettement plus coûteuse qu'une recherche directement exploitable par un index.

## 4. Solution 1 : ne pas transformer la colonne

Lorsque c'est possible, il est préférable que la colonne apparaisse directement dans la condition :

```sql
SELECT id, email, name
FROM users
WHERE email = 'john@example.com';
```

La condition est alors de la forme :

```sql
colonne = valeur
```

MySQL peut beaucoup plus facilement utiliser l'index sur `email` pour rechercher directement la valeur souhaitée. Dans le cas des chaînes de caractères, il faut toutefois tenir compte de la collation. Avec une collation insensible à la casse, courante dans MySQL, l'utilisation de `LOWER()` peut être inutile :

```sql
WHERE email = 'john@example.com'
```

peut déjà correspondre à :

```text
john@example.com
John@example.com
JOHN@EXAMPLE.COM
```

selon la collation de `email`.

## 5. Solution 2 : utiliser un index fonctionnel

Si l'application de `LOWER()` est réellement nécessaire, les versions modernes de MySQL permettent notamment d'indexer une expression :

```sql
CREATE INDEX idx_users_lower_email
ON users ((LOWER(email)));
```

Une requête utilisant la même expression peut alors bénéficier de cet index :

```sql
SELECT id, email, name
FROM users
WHERE LOWER(email) = 'john@example.com';
```

On parle alors d'**index fonctionnel**.

## 6. Le problème ne concerne pas uniquement `LOWER()`

Le même principe s'applique à de nombreuses fonctions ou expressions appliquées à une colonne indexée. Exemple :

```sql
WHERE YEAR(created_at) = 2026
```

peut être moins efficace que :

```sql
WHERE created_at >= '2026-01-01'
  AND created_at <  '2027-01-01'
```

La seconde écriture est particulièrement intéressante si `created_at` est indexée, car MySQL peut effectuer une **recherche par intervalle dans l'index**.

Même problématique avec :

```sql
WHERE UPPER(username) = 'DUPONT'

WHERE DATE(created_at) = '2026-10-06'

WHERE SUBSTRING(phone, 1, 2) = '06'

WHERE ABS(amount) > 100
```

Cela ne signifie pas que les fonctions SQL sont mauvaises ou qu'il ne faut jamais les utiliser dans `WHERE`. Le problème apparaît surtout lorsque la transformation de la colonne empêche le moteur d'utiliser efficacement un index existant.

## 7. Vérifier plutôt que supposer

Le comportement exact dépend de MySQL, de sa version, des index, de la collation, des statistiques et de la requête. Il est donc préférable de vérifier le plan d'exécution :

```sql
EXPLAIN
  SELECT *
  FROM users
  WHERE LOWER(email) = 'john@example.com';
```

puis de le comparer avec :

```sql
EXPLAIN
  SELECT *
  FROM users
  WHERE email = 'john@example.com';
```

À retenir : lorsqu'une colonne indexée intervient dans un `WHERE`, privilégiez lorsque c'est possible une condition permettant au moteur de travailler directement sur la colonne :

```sql
WHERE colonne = valeur
WHERE colonne > valeur
WHERE colonne BETWEEN valeur1 AND valeur2
```

plutôt qu'une transformation :

```sql
WHERE FONCTION(colonne) = valeur
```

L'objectif est de conserver une condition "sargable", afin de permettre à MySQL d'exploiter efficacement ses index.
