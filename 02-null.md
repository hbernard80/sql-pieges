# 10 pièges SQL (MySQL) et plus > 2 - Utilisation de `IS NULL` ou `IS NOT NULL`

![images/sql-logo.png](images/sql-logo.png)

## 1. Requête

```sql
SELECT id, label, description, sku, price
FROM products
WHERE price = NULL;
```

## 2. Problèmes

La requête ne retourne aucune ligne, même s'il existe des produits dont le prix vaut `NULL`.

## 3. Pourquoi

`colonne = NULL` ne donne jamais `TRUE` : en SQL, le résultat de cette comparaison est `UNKNOWN` (inconnu). Or la clause `WHERE` ne conserve que les lignes pour lesquelles la condition vaut `TRUE`.

## 4. Solution

Utilisez `IS NULL`, ou `IS NOT NULL` pour le contraire (au lieu de `!= NULL` ou `<> NULL`) :

```sql
SELECT id, label, description, sku, price
FROM products
WHERE price IS NULL;
```

Pour afficher les produits qui ont bien un prix renseigné :

```sql
SELECT id, label, description, sku, price
FROM products
WHERE price IS NOT NULL;
```

> 💡 Pour un prix non renseigné, il vaut mieux mettre `NULL`. La valeur _0.00_ veut dire : _le prix est connu, et il est égal à zéro_. `NULL` signifie : _le prix est inconnu / non renseigné_. En outre, un prix ne devrait pas être égal à zéro, c'est-à-dire gratuit : pour éviter cela il est recommandée de placer une contrainte _> 0_ sur la colonne.
