# 10 pièges SQL (MySQL) et plus > 10 - `OFFSET` trop élevé

![images/sql-logo.png](images/sql-logo.png)
  
## 1. Le problème de `OFFSET`

La pagination classique en SQL repose souvent sur l'association de `LIMIT` et `OFFSET` :

```sql
SELECT *
FROM orders
ORDER BY id
LIMIT 20 OFFSET 100000;
```

Cette requête signifie : _Ignore les 100 000 premières lignes du résultat trié, puis retourne les 20 suivantes_. Cette technique est simple et parfaitement acceptable pour les premières pages. Elle peut cependant devenir coûteuse lorsque la valeur de `OFFSET` devient importante. `OFFSET` ne permet pas à MySQL de se rendre directement à la 100 001e ligne du résultat comme on se déplacerait à une position précise dans un tableau.

Pour produire le résultat, MySQL doit généralement traiter les lignes précédentes afin de déterminer lesquelles doivent être ignorées. Avec :

```sql
LIMIT 20 OFFSET 100000
```

le moteur doit donc atteindre 100 020 lignes, puis écarter les 100 000 premières pour n'en retourner que 20.

Plus l'utilisateur avance dans les pages, plus le travail peut donc augmenter :

```text
Page 1       OFFSET 0
Page 10      OFFSET 180
Page 1 000   OFFSET 19 980
Page 5 001   OFFSET 100 000
```

Sur une table volumineuse, une pagination profonde peut ainsi entraîner une augmentation du temps de réponse et de la quantité de données que MySQL doit examiner. Un index adapté à `ORDER BY` peut considérablement améliorer la requête, mais il ne supprime pas nécessairement le coût lié au parcours des entrées précédant l'offset.

## 2. La solution : la keyset pagination

Lorsque l'application n'a pas besoin d'accéder directement à une page arbitraire, on peut utiliser la keyset pagination, également appelée pagination par curseur (ou _seek pagination_).

Au lieu de demander : _Donne-moi 20 lignes après avoir ignoré les 100 000 premières_, on demande _Donne-moi les 20 lignes situées après la dernière ligne que j'ai déjà reçue_.

Si la dernière commande affichée possède l'identifiant `100000` :

```sql
SELECT *
FROM orders
WHERE id > 100000
ORDER BY id
LIMIT 20;
```

Avec un index sur `id` — ce qui est normalement le cas s'il s'agit de la clé primaire — MySQL peut rechercher directement les valeurs supérieures à `100000`, sans avoir à parcourir les 100 000 résultats précédents de la pagination.

## 3. Exemple concret

Première page :

```sql
SELECT *
FROM orders
ORDER BY id
LIMIT 20;
```

Supposons que le dernier résultat possède :

```text
id = 1528
```

Pour obtenir la page suivante :

```sql
SELECT *
FROM orders
WHERE id > 1528
ORDER BY id
LIMIT 20;
```

Si cette nouvelle page se termine avec `id = 1563`, la requête suivante sera :

```sql
SELECT *
FROM orders
WHERE id > 1563
ORDER BY id
LIMIT 20;
```

On ne calcule donc plus la position de la page. On conserve simplement la dernière valeur de tri obtenue, qui joue le rôle de curseur.

## 4. Attention au critère de tri

La keyset pagination fonctionne particulièrement simplement avec une colonne unique, indexée et permettant un ordre déterministe, telle qu'une clé primaire.

Avec un tri sur une colonne qui peut contenir des doublons, par exemple :

```sql
ORDER BY created_at
```

utiliser uniquement :

```sql
WHERE created_at > :last_created_at
```

peut être insuffisant : plusieurs commandes peuvent avoir exactement la même date. On peut alors ajouter l'identifiant comme second critère :

```sql
SELECT *
FROM orders
WHERE created_at > :last_created_at
   OR (created_at = :last_created_at AND id > :last_id)
ORDER BY created_at, id
LIMIT 20;
```

Un index composite adapté sera alors particulièrement intéressant :

```sql
CREATE INDEX idx_orders_created_at_id
  ON orders (created_at, id);
```

Le couple `(created_at, id)` constitue ici le curseur.

## 5. `OFFSET` ou keyset pagination ?

| Besoin | Technique adaptée |
| ------ | ----------------- |
| Quelques pages seulement | `LIMIT / OFFSET` |
| Petit volume de données | `LIMIT / OFFSET` |
| Accéder directement à la page 42 | `LIMIT / OFFSET` |
| Très grand nombre de lignes | **Keyset pagination** |
| Scroll infini | **Keyset pagination** |
| API avec `next_cursor` | **Keyset pagination** |
| Pagination très profonde | **Keyset pagination** |
| Performances régulières quelle que soit la profondeur | **Keyset pagination** |

La contrepartie principale de la keyset pagination est qu'elle se prête moins bien à une interface proposant _Aller directement à la page 5 000_. Elle est surtout adaptée à une navigation suivant/précédent, à un scroll infini ou à une API utilisant des curseurs.

**À retenir :** `LIMIT` n'est pas le problème en soi. C'est surtout un `OFFSET` très élevé qui peut devenir coûteux. Pour les gros volumes et les paginations profondes, la keyset pagination permet au SGBD de repartir d'une valeur connue et indexée plutôt que d'ignorer un nombre croissant de résultats.
