# 10 pièges SQL (MySQL) et plus > 3 - Les fonctions `MAX()` et `MIN()`

![images/sql-logo.png](images/sql-logo.png)

## 1. Requête

Trouver les produits qui ont le prix le plus élevé :

```sql
SELECT id, label, description, sku, price
FROM products
WHERE MAX(price);
```

## 2. Problème

`MAX()` et `MIN()` sont des fonctions d'agrégation. Elle ne s'utilisent pas directement dans la clause `WHERE`, qui filtre les lignes avant le calcul des agrégats.

## 3. Solutions

Pour corriger cela, la tentation est grande de mettre la fonction MAX dans l'instruction `SELECT` (liste des colonnes) :

```sql
SELECT id, label, description, sku, MAX(price)
FROM products;
```

Mais le SQL ne sortira pas forcément les autres colonnes (`id`, `label`, `description`, `sku`) associées à la ligne qui a effectivement le prix le plus élevé , contrairement à ce que le tableau de résultats laisse penser, ce qui est traître.  

La bonne solution est l'utilisation d'une sous-requête :

```sql
SELECT id, label, description, sku, price
FROM products
WHERE price = (SELECT MAX(price) FROM products);
```

Les valeurs affichées sur une ligne correspondront bien au produit le plus cher, et, autre avantage non négligeable, cela retournera tous les produits qui coûtent ce prix, dans le cas où il y en a plusieurs :

| label | price |
| ----- | ---- |
| A | 100 |
| B | 50 |
| C | 100 |

Ici les produits A et C seront retournés.

## 4. Même problème avec un `GROUP BY`

Exemple :

```sql
SELECT order_id, customer_id, MAX(order_date)
FROM orders
GROUP BY customer_id;
```

De nombreux développeurs pensent récupérer la dernière commande de chaque client. En réalité, l'`id` retourné n'est pas garanti comme étant celui de la commande la plus récente.

## 5. Autres problèmes avec `MAX()` et `MIN()`

### 5.a. Comparaison alphabétique sur les colonnes texte

Exemple :

| valeur |
| ------ |
| 1 |
| 2 |
| 10 |

Si la colonne est de type `VARCHAR` :

```sql
SELECT MAX(valeur)
FROM table;
```

Résultat :

```text
2
```

et non :

```text
10
```

car MySQL compare les chaînes de caractères alphabétiquement.
