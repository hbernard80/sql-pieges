# 10 pièges SQL (MySQL) et plus > 1 - `LIMIT` sans `ORDER BY`

![images/sql-logo.png](images/sql-logo.png)

## 1. La requête

```sql
SELECT id, label, description, sku, price
FROM products
LIMIT 5;
```

## 2. Le problème

La requête retourne au maximum 5 lignes, mais pas les 5 premières : sans `ORDER BY`, SQL ne garantit ni les lignes retournées ni leur ordre. Le résultat peut varier selon le plan d'exécution, les index utilisés ou les modifications apportées à la table. Ceci est valable pour tous les <abbr title="Système de Gestion de Bases de Données Relationnelles">SGBDR</abbr>.

> [L'instruction `EXPLAIN`](explain.md) permet une analyse du plan d'exécution.

## 3. Pourquoi ?

Une table relationnelle représente un ensemble (= _set_) de tuples. Or, par définition, un ensemble n'a pas d'ordre. Ainsi, la requête (ou `TOP 5` sous SQL Server), signifie simplement _Retourner 5 lignes du résultat_. Le standard SQL ne précise jamais lesquelles.

## 3. Pourquoi parle-t-on aussi des lignes retournées ?

Prenons une table contenant 1 000 lignes. On pourrait croire que seules les 5 premières lignes sont ambiguës et que les lignes elles-mêmes sont fixes.

En réalité, le moteur est libre de :

* parcourir la table physique ;
* parcourir un index ;
* utiliser un index couvrant ;
* utiliser un parcours parallèle ;
* changer complètement de plan d'exécution.

Ainsi, deux exécutions peuvent renvoyer :

Exécution 1

| id |
| -- |
| 1  |
| 2  |
| 3  |
| 4  |
| 5  |

Exécution 2

| id  |
| --- |
| 54  |
| 61  |
| 72  |
| 85  |
| 101 |

Les lignes retournées peuvent donc elles-mêmes être différentes.

## 4. Pourquoi observe-t-on souvent toujours le même résultat ?

Parce que les SGBDR sont généralement déterministes tant que :

* les données ne changent pas
* les statistiques ne changent pas
* les index restent identiques
* le plan d'exécution reste le même.

Par exemple, avec le moteur de stockage _InnoDB_ sous MySQL, la requête renverra souvent les 5 plus petites clés primaires, car les données sont stockées dans un index clusterisé.

* Sous PostgreSQL, on obtient souvent l'ordre physique de stockage (_heap_).
* Sous SQL Server, on voit souvent également un ordre stable.

Mais il s'agit uniquement d'un effet de l'implémentation, pas d'une garantie.

## 5. Une formulation encore plus rigoureuse

Sans `ORDER BY`, le standard SQL ne garantit ni l'ordre du résultat ni l'identité des lignes retournées. Un SGBDR peut produire un résultat stable pendant des années, puis en produire un différent après une modification des données, des index, des statistiques ou du plan d'exécution. Il ne faut donc jamais s'appuyer sur l'ordre apparent obtenu sans `ORDER BY`.

## 6. Solution

Spécifiez un `ORDER BY` sur l'une des colonnes, en fonction de la requête métier demandée, par exemple _afficher les 5 produits les moins chers_, donc un tri croissant sur le prix :

```sql
SELECT id, label, description, sku, price
FROM products
ORDER BY price ASC, id ASC
LIMIT 5;
```

Précision : ici, l'ajout de `id ASC` permet d'afficher les produits qui ont le même prix.
