# Antisèche oral, Elasticsearch / OpenSearch

## Les 5 pièges (si tu retiens que ça, retiens ça)
1. Sqoop n'est pas fait pour les logs. C'est du SQL vers Hadoop en batch (et c'est retiré). Les logs passent par Logstash, Beats, Kafka ou Data Prepper.
2. Licences 2026 : ES est redevenu open source en 2024 (AGPLv3). La vraie différence, c'est la gouvernance (OpenSearch est en Apache 2.0, sous la Linux Foundation).
3. term contre match : non-analysé contre analysé. Pas "structuré contre full-text".
4. Agrégations : ça lit les `doc_values` (stockage en colonne, sur disque ou off-heap), pas l'index inversé.
5. Index inversé contre SQL : SQL a aussi des index. La vraie différence c'est le full-text, retrouver un mot au milieu d'un texte.

---

## Pitch, le fil conducteur (7 min)
- Accroche (45s) : 3h du matin, panne, des millions de logs. Stocker, ce n'est pas chercher instantanément.
- ES (65s) : 2010, Shay Banon, les recettes de cuisine. Java plus Lucene, API REST/JSON. Fork OpenSearch en 2021, même ADN.
- Archi (95s), trois idées : cluster, nœuds et shards pour l'échelle. Réplicas pour la résilience. Et l'index inversé, comme l'index d'un livre, c'est le secret de la vitesse.
- Big Data (95s), le voyage de la donnée : ingestion, puis HDFS/Spark/Hive en batch, puis MongoDB/Redis/Neo4j, et ES comme couche recherche sur la donnée chaude. À retenir : complémentaires, pas concurrents.
- NoSQL (40s) : Mongo pour le stockage, Cassandra en query-first, Neo4j pour les relations. Le bon outil pour le bon besoin.
- Démo (35s) : 14 000 logs, 2 requêtes structurées (404, dates), 2 en full-text (error, opensearch), 1 dashboard.

---

## Réponses express (Q/R)

Index inversé ? Un mot pointe vers la liste des docs. C'est pré-payé à l'écriture. Le `LIKE '%x%'` en SQL fait un scan complet.

Analyzer ? Il met en minuscule et tokenise. `match` analyse, `term` non. Du coup `term "Error"` renvoie 0 résultat.

Limites d'ES ? Pas une source de vérité, pas d'ACID multi-docs, near-real-time, heap à 32 Go max, jointures faibles.

refresh / flush ? refresh rend cherchable (environ 1s). La durabilité passe par le translog. flush écrit durablement sur disque.

master contre data ? master gère l'état du cluster (pas la donnée). data stocke et exécute. Et il y a aussi ingest et coordinating.

Split-brain ? Une partition réseau donne 2 masters, d'où le quorum à (N/2)+1. Avant ES7 c'était `minimum_master_nodes` à la main. ES7 et après : voting config en auto.

Primaire contre réplica ? Le primaire vient de `hash(_id) % nb`, figé. Le réplica est une copie (HA et lecture), modifiable à chaud.

Un nœud tombe ? Avec au moins 1 réplica, il est promu, on passe de jaune à vert, pas de perte. Avec 0 réplica, c'est rouge et il y a perte.

Pourquoi pas Mongo ? Le full-text n'est pas son cœur de métier (et Atlas Search, c'est du Lucene quand même).

Pourquoi pas Cassandra ? Query-first, il faut connaître la requête à l'avance.

Pourquoi pas Neo4j ? Le graphe sert pour les relations, pas pour des logs plats.

Pourquoi pas Redis ? Clé-valeur en RAM, il n'indexe pas le contenu.

ES = une base ? Non, c'est un moteur de recherche posé au-dessus d'une base de vérité.

Pourquoi pas tout Hadoop ? Hadoop c'est du batch (minutes ou heures), ES c'est du near-real-time (ms ou s).

Sqoop ingère les logs ? Non. C'est Logstash, Beats ou Kafka.

ELK ? Elasticsearch plus Logstash plus Kibana, qui devient OpenSearch plus Data Prepper plus OpenSearch Dashboards.

Pourquoi Hadoop ET ES ? Le chaud (ES, cher mais rapide) contre le froid (HDFS, bas coût mais lent).

ES open source ? Apache 2.0 (2010 à 21), puis SSPL/ELv2 (21 à 24), puis ajout d'AGPL OSI en 2024. OpenSearch reste en Apache 2.0 depuis toujours.

Licences ? Apache 2.0 et AGPL sont libres. SSPL et Elastic License v2 sont source-available (pas OSI).

Gratuit = sans coût ? Le logiciel oui, l'exploitation non (RAM, réplicas qui doublent, CPU, ops).

RAM ? La moitié au heap, 31 Go max environ (compressed oops).

Le risque numéro 1 ? Un mauvais sharding. L'over-sharding fait grimper le heap, et c'est figé après la création.

term contre match en interne ? term est en filter context, sans score, cacheable. match est en query context, avec BM25, et c'est trié.

term sur du texte = 0 ? Pas d'analyzer, donc le token "error" n'égale pas "Error". La solution c'est `.keyword`.

multi_match qui renvoie 5101, c'est du bruit ? Non, ce n'est pas faux, c'est trié par score donc les pertinents sont en haut. Pour réduire : boost, minimum_score.

filter context ? Pour les logs répétitifs (range, term), ça donne un bitset caché, quasi gratuit. Le match va dans `must` uniquement.

Coût d'une agg terms ? Ça lit les doc_values. `size:0` pour ne pas avoir de hits. Le danger, c'est la forte cardinalité (une ip par exemple).

Rapide à 14M ? Index inversé, plus sharding (donc parallèle), plus réplicas. Rien de magique : ça dépend du sharding.

---
*Respire. Souris. "Complémentaires, pas concurrents." Bonne chance.*
