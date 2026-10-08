# 10 pièges SQL (MySQL) et plus > 6 - Fonctions sur jokers (`LIKE '%recherche'`)

![images/sql-logo.png](images/sql-logo.png)

## 1. `LIKE '%recherche'` ou `LIKE '%recherche%'`

La clause `LIKE` permet d'effectuer une recherche selon un motif dans une colonne textuelle. Le caractère `%` est un joker qui représente zéro, un ou plusieurs caractères :

```sql
SELECT *
FROM products
WHERE label LIKE 'iphone%';
```

Cette requête recherche toutes les valeurs qui commencent par `iphone`. Elle peut notamment retourner :

* `iphone`
* `iphone 15`
* `iphone pro`
* `iphone reconditionné`

À l'inverse :

```sql
SELECT *
FROM products
WHERE label LIKE '%iphone';
```

recherche les valeurs qui se terminent par `iphone`.

Enfin :

```sql
SELECT *
FROM products
WHERE label LIKE '%iphone%';
```

recherche les valeurs qui contiennent `iphone, quelle que soit sa position.

## 2. Le problème du joker `%` placé au début

Le principal problème de performances apparaît lorsque le motif commence par `%` :

```sql
WHERE label LIKE '%iphone'
```

ou :

```sql
WHERE label LIKE '%iphone%'
```

Même si la colonne `label` possède un index classique :

```sql
CREATE INDEX idx_products_label
ON products(label);
```

MySQL ne peut généralement pas utiliser efficacement cet index B-tree pour déterminer directement où commencer la recherche.

### 2.a. Pourquoi l'index devient-il inefficace ?

Un index B-tree conserve les valeurs dans un ordre permettant de rechercher efficacement une valeur ou un préfixe connu.

Par exemple, avec :

```sql
WHERE label LIKE 'iphone%'
```

MySQL connaît le début de la chaîne recherchée : `iphone`.

Il peut donc localiser dans l'index la zone contenant les valeurs commençant par `iphone`, puis parcourir uniquement cette partie de l'index.

On peut schématiser l'index ainsi :

```text
ipad
ipad air
iphone
iphone 15
iphone 16
iphone pro
macbook
pixel
samsung
```

Avec :

```sql
LIKE 'iphone%'
```

MySQL sait qu'il doit commencer approximativement à :

```text
iphone
↑
```

et peut arrêter son parcours lorsque les valeurs ne commencent plus par `iphone`. En revanche, avec :

```sql
LIKE '%iphone%'
```

le début de la valeur est inconnu.

`iphone` pourrait se trouver dans :

```text
iphone 16
apple iphone
coque iphone
occasion iphone 15
smartphone iphone reconditionné
```

L'ordre alphabétique de l'index ne permet donc plus de localiser directement une plage pertinente. MySQL doit potentiellement examiner un très grand nombre de lignes ou d'entrées d'index pour vérifier le motif.

## 3. Différence entre les principaux motifs `LIKE`

Les performances ne sont donc pas identiques selon la position du joker.

| Condition | Signification | Index B-tree potentiellement efficace |
| --------- | ------------- | ------------------------------------- |
| `LIKE 'iphone'` | valeur exacte | Oui |
| `LIKE 'iphone%'` | commence par `iphone` | Oui |
| `LIKE '%iphone'` | se termine par `iphone` | Généralement non pour localiser directement les lignes |
| `LIKE '%iphone%'` | contient `iphone` | Généralement non pour localiser directement les lignes |

La règle importante est donc : _Un joker `%` placé au début du motif empêche généralement MySQL d'effectuer une recherche par plage efficace dans un index B-tree classique_.

## 4. Exemple problématique

Considérons une table contenant plusieurs millions de produits :

```sql
SELECT *
FROM products
WHERE label LIKE '%iphone%';
```

Cette requête présente deux problèmes :

1. Le motif commence par `%`. MySQL ne peut donc pas simplement rechercher dans l'index les valeurs commençant par `iphone`.
2. `SELECT *` demande le chargement de toutes les colonnes des produits trouvés, même lorsque certaines ne sont pas nécessaires.

Sur une petite table, cela peut rester imperceptible. Sur plusieurs millions de lignes, le coût peut devenir important.

## 5. Première optimisation : supprimer le joker initial lorsque c'est possible

Si le besoin fonctionnel consiste uniquement à rechercher les produits dont le nom **commence par** le texte saisi, il vaut mieux écrire :

```sql
SELECT id, label, description, sku, price
FROM products
WHERE label LIKE 'iphone%';
```

Avec un index approprié :

```sql
CREATE INDEX idx_products_label
ON products(label);
```

MySQL peut potentiellement effectuer une recherche par plage dans l'index. Cette solution est particulièrement adaptée à des fonctionnalités telles que :

* l'autocomplétion ;
* la recherche par préfixe ;
* la recherche d'une référence ;
* la recherche d'un nom commençant par une chaîne donnée.

Par exemple, une saisie utilisateur :

```text
iph
```

peut produire :

```sql
WHERE label LIKE 'iph%'
```

et retrouver :

```text
iphone
iphone 15
iphone 16 pro
```

## 6. Vérifier réellement l'utilisation de l'index avec `EXPLAIN`

Il ne faut pas supposer qu'un index sera nécessairement utilisé. Il faut vérifier le plan d'exécution.

Par exemple :

```sql
EXPLAIN
SELECT id, label
FROM products
WHERE label LIKE 'iphone%';
```

Puis comparer avec :

```sql
EXPLAIN
SELECT id, label
FROM products
WHERE label LIKE '%iphone%';
```

Selon la version de MySQL, on peut également utiliser :

```sql
EXPLAIN ANALYZE
SELECT id, label
FROM products
WHERE label LIKE '%iphone%';
```

`EXPLAIN ANALYZE` permet d'obtenir des informations sur l'exécution réelle de la requête, notamment le nombre de lignes effectivement parcourues et le temps consacré aux différentes opérations.

Il est donc particulièrement utile pour confirmer qu'une optimisation apporte réellement un gain.

## 7. Utiliser un index `FULLTEXT` pour une véritable recherche textuelle

Lorsque le besoin fonctionnel consiste réellement à effectuer une recherche dans du texte, transformer :

```sql
LIKE '%iphone%'
```

en :

```sql
LIKE 'iphone%'
```

n'est pas toujours possible. Les deux requêtes ne répondent pas au même besoin. Pour rechercher des mots dans des champs textuels, MySQL propose les index `FULLTEXT`.

Par exemple :

```sql
CREATE FULLTEXT INDEX ft_products_label
ON products(label);
```

La recherche peut ensuite être effectuée avec :

```sql
SELECT id, label, description, sku, price
FROM products
WHERE MATCH(label) AGAINST('iphone');
```

Il est également possible d'indexer plusieurs colonnes :

```sql
CREATE FULLTEXT INDEX ft_products_search
ON products(label, description);
```

puis :

```sql
SELECT id, label, description, sku, price
FROM products
WHERE MATCH(label, description)
      AGAINST('iphone');
```

Cette approche est généralement mieux adaptée à une fonctionnalité de recherche textuelle sur un volume important de données.

## 8. `FULLTEXT` n'est  pas un remplacement exact de `LIKE '%mot%'`

`FULLTEXT` n'est toutefois pas un remplacement exact de `LIKE '%mot%'`. Il faut faire attention à cette différence importante.

Une recherche :

```sql
LIKE '%phone%'
```

cherche une suite de caractères.

Elle peut donc retrouver `iphone` parce que la chaîne `phone` est présente à l'intérieur du mot. Une recherche `FULLTEXT`, en revanche, travaille principalement sur des termes et des mots indexés.

Ainsi :

```sql
MATCH(label) AGAINST('phone')
```

ne doit pas être considéré comme strictement équivalent à :

```sql
label LIKE '%phone%'
```

Le choix dépend donc du besoin fonctionnel.

* Pour rechercher un préfixe** : `LIKE 'mot%'` est souvent approprié.
* Pour rechercher une sous-chaîne arbitraire : `LIKE '%mot%'` répond directement au besoin, mais peut être coûteux.
* Pour effectuer une recherche textuelle par mots sur de gros volumes : un index `FULLTEXT` est généralement plus adapté.

## 9. Attention aux optimisations qui modifient le résultat

Il serait incorrect de présenter systématiquement :

```sql
WHERE label LIKE 'iphone%'
```

comme une simple optimisation de :

```sql
WHERE label LIKE '%iphone%'
```

Les deux requêtes n'ont pas la même signification. Prenons les données suivantes :

```text
iphone 16
iphone reconditionné
apple iphone 15
coque pour iphone
```

Avec :

```sql
LIKE 'iphone%'
```

on retrouvera principalement :

```text
iphone 16
iphone reconditionné
```

Alors qu'avec :

```sql
LIKE '%iphone%'
```

les quatre valeurs correspondent au motif. Une optimisation SQL ne doit donc pas modifier le comportement fonctionnel attendu uniquement pour améliorer les performances.

## 10. Bonnes pratiques

Avant d'utiliser :

```sql
LIKE '%recherche%'
```

sur une table volumineuse, il faut se poser plusieurs questions :

* A-t-on réellement besoin d'une recherche de sous-chaîne ?
* Une recherche par préfixe avec `LIKE 'recherche%'` serait-elle suffisante ?
* La colonne possède-t-elle un index adapté ?
* Un index `FULLTEXT` correspond-il mieux au besoin ?
* La requête sélectionne-t-elle uniquement les colonnes nécessaires ?
* Le comportement réel a-t-il été vérifié avec `EXPLAIN` ou `EXPLAIN ANALYZE` ?
* La volumétrie justifie-t-elle une solution de recherche spécialisée ?

Pour une application possédant une recherche avancée, avec de très gros volumes, de la pertinence, de la tolérance aux fautes ou des fonctionnalités complexes, un moteur spécialisé comme Elasticsearch, OpenSearch ou Meilisearch peut également devenir plus approprié qu'une succession de conditions `LIKE`.

## 11. À retenir

Le point essentiel n'est pas que `LIKE` soit intrinsèquement lent. Une condition telle que :

```sql
WHERE label LIKE 'iphone%'
```

peut être performante avec un index B-tree approprié. Le problème concerne surtout les recherches dont le motif commence par un joker :

```sql
WHERE label LIKE '%iphone'
```

et :

```sql
WHERE label LIKE '%iphone%'
```

Dans ces situations, MySQL ne dispose pas d'un préfixe connu lui permettant de localiser directement une plage de valeurs dans un index B-tree classique.

Sur une petite table, cela n'est pas nécessairement problématique. Sur une table contenant des millions de lignes et exécutant fréquemment ce type de recherche, le coût peut devenir considérable.

La bonne démarche consiste donc à choisir la technique en fonction du besoin :

```text
Recherche exacte
        ↓
= 'iphone'

Recherche par préfixe
        ↓
LIKE 'iphone%'

Recherche de sous-chaîne
        ↓
LIKE '%iphone%'
(coût à surveiller)

Recherche textuelle par mots
        ↓
FULLTEXT

Recherche avancée à grande échelle
        ↓
Moteur de recherche spécialisé
```
