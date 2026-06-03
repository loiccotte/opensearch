# 🎯 ANTISÈCHE ORAL — Elasticsearch / OpenSearch

## ⛔ LES 5 PIÈGES (si tu retiens que ça, retiens ça)
1. **Sqoop ≠ logs** → SQL↔Hadoop batch (retiré). Logs = Logstash/Beats/Kafka/Data Prepper
2. **Licences 2026** → ES **redevenu open source** en 2024 (AGPLv3). Diff = gouvernance (OpenSearch = Apache 2.0 / Linux Foundation)
3. **term vs match** = non-analysé vs **analysé** (pas « structuré vs full-text »)
4. **Agrégations** = `doc_values` (colonne, disque/off-heap), **pas** l'index inversé
5. **Index inversé vs SQL** → SQL a des index ; la diff c'est le **full-text** (mot au milieu d'un texte)

---

## 🎤 PITCH — fil conducteur (7 min)
- **① Accroche (45s)** — 3h du matin, panne, millions de logs → STOCKER ≠ CHERCHER instantanément
- **② ES (65s)** — 2010, Shay Banon, recettes de cuisine · Java + **Lucene** · API REST/JSON · **fork OpenSearch 2021, même ADN**
- **③ Archi (95s)** — *3 idées* : cluster/nœuds/shards (échelle) · réplicas (résilience) · **INDEX INVERSÉ = index d'un livre** (le secret de la vitesse)
- **④ Big Data (95s)** — voyage de la donnée : ingestion → HDFS/Spark/Hive (batch) → MongoDB/Redis/Neo4j → **ES = couche recherche, donnée chaude**. ⭐ **COMPLÉMENTAIRES, PAS CONCURRENTS**
- **⑤ NoSQL (40s)** — Mongo (stockage), Cassandra (query-first), Neo4j (relations) → « le bon outil pour le bon besoin »
- **⑥ Démo (35s)** — 14 000 logs · 2 structurées (404, dates) · 2 full-text (error, opensearch) · 1 dashboard

---

## 🛡️ RÉPONSES EXPRESS (Q/R)

**Index inversé ?** mot → liste des docs. Pré-payé à l'écriture. SQL `LIKE '%x%'` = scan complet.
**Analyzer ?** minuscule + tokenise. `match` analyse, `term` non → `term "Error"` = 0 résultat.
**Limites ES ?** pas source de vérité, pas d'ACID multi-docs, near-real-time, heap ≤32 Go, jointures faibles.
**refresh/flush ?** refresh = cherchable (~1s) · durabilité = **translog** · flush = durable sur disque.
**master vs data ?** master = état du cluster (PAS la donnée) · data = stocke+exécute · + ingest, coordinating.
**Split-brain ?** partition → 2 masters → quorum (N/2)+1. Avant ES7 = `minimum_master_nodes` manuel · ES7+ = voting config **auto**.
**Primaire vs réplica ?** primaire = `hash(_id) % nb` → figé · réplica = copie (HA + lecture), modifiable à chaud.
**Nœud tombe ?** ≥1 réplica → promu, JAUNE→VERT, pas de perte · 0 réplica → ROUGE, perte.

**Pourquoi pas Mongo ?** full-text pas son cœur (Atlas Search = Lucene quand même).
**Pourquoi pas Cassandra ?** query-first, faut connaître la requête d'avance.
**Pourquoi pas Neo4j ?** graphe = relations, pas des logs plats.
**Pourquoi pas Redis ?** clé-valeur RAM, n'indexe pas le contenu.
**ES = base ?** non, moteur de recherche **au-dessus** d'une base de vérité.

**Pourquoi pas tout Hadoop ?** Hadoop = batch (min/h) · ES = near-real-time (ms/s).
**Sqoop ingère les logs ?** ⛔ NON → Logstash/Beats/Kafka.
**ELK ?** Elasticsearch + Logstash + Kibana → OpenSearch + Data Prepper + OpenSearch Dashboards.
**Pourquoi Hadoop ET ES ?** chaud (ES, cher/rapide) vs froid (HDFS, bas coût/lent).

**ES open source ?** Apache 2.0 (2010-21) → SSPL/ELv2 (21-24) → **+AGPL OSI (2024)**. OpenSearch = Apache 2.0 tjs.
**Licences ?** Apache 2.0 & AGPL = libres · SSPL & Elastic License v2 = source-available (pas OSI).
**Gratuit = sans coût ?** logiciel oui, exploit non (RAM, réplicas ×2, CPU, ops).
**RAM ?** 50% au heap, max ~31 Go (compressed oops).
**Risque n°1 ?** mauvais sharding (over-sharding = surcoût heap, figé après création).

**term vs match interne ?** term = filter context, pas de score, cacheable · match = query context, **BM25**, trié.
**term sur texte = 0 ?** pas d'analyzer → token « error » ≠ « Error ». Solution : `.keyword`.
**multi_match 5101 = bruit ?** non faux mais trié par score, les pertinents en haut · réduire : boost, minimum_score.
**filter context ?** logs répétitifs (range, term) → bitset caché, quasi gratuit · match dans `must` seulement.
**agg terms coût ?** lit doc_values · `size:0` = pas de hits · danger = **forte cardinalité** (ip).
**rapide à 14M ?** index inversé + sharding (parallèle) + réplicas. Pas magique : dépend du sharding.

---
*Respire. Souris. « Complémentaires, pas concurrents ». Bonne chance.*
