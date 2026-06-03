# Préparation Oral CC2 — Elasticsearch / OpenSearch

> 7 min présentation + 3 min Q/R · examinateur strict · binôme
> Ce document = aide de préparation (le pitch à dire + les enjeux + les questions pièges avec réponses).

---

## ⚠️ LES 5 CORRECTIONS CRITIQUES (à connaître absolument)

Le prof va taper exactement là. Ce sont des erreurs présentes dans les slides/discours par défaut :

1. **Sqoop ≠ ingestion de logs.** Sqoop = transfert **batch SQL ↔ Hadoop** (JDBC, données structurées). C'est même un projet *retiré* (Apache Attic). Pour ingérer des **logs** → Logstash / Beats / Fluentd / **Data Prepper** (côté OpenSearch) / **Kafka**. Sqoop est sur ta slide 2.1 → piège quasi garanti.

2. **Licences en 2026 :** « OpenSearch est libre, Elasticsearch ne l'est plus » est **PÉRIMÉ**. Depuis **août 2024 (v8.16, licence AGPLv3)**, Elasticsearch est **redevenu open source**. Aujourd'hui la différence est surtout de **gouvernance** (OpenSearch = Apache 2.0, sous la **Linux Foundation** depuis 2024).

3. **term vs match = « non-analysé vs analysé »**, PAS « structuré vs full-text ». Un `term` ne passe **jamais** par l'analyzer → un `term` sur un champ `text` peut renvoyer **0 résultat** à cause de la casse (le token stocké est en minuscule).

4. **Les agrégations utilisent les `doc_values`** (stockage **colonne**, sur **disque / off-heap**), **PAS l'index inversé**. Et les doc_values **ne consomment pas le heap** (contrairement à l'ancien fielddata).

5. **Index inversé vs SQL :** ne JAMAIS dire « SQL parcourt tout ligne par ligne ». SQL a des index B-tree. La vraie différence = le **full-text** : chercher un mot **au milieu** d'un texte (un `LIKE '%mot%'` SQL, lui, scanne tout).

---

# PARTIE 1 — LE PITCH (7 min · ~420s)

## ① Accroche (45s)
> Imaginez : il est trois heures du matin, votre site e-commerce ne répond plus, et quelque part dans vos serveurs, plusieurs millions de lignes de logs cachent LA ligne qui explique la panne. Avec une base de données classique, retrouver cette ligne, c'est chercher une aiguille dans une botte de foin — plusieurs minutes, parfois des heures. Or quand un site est à terre, chaque minute coûte de l'argent. Le vrai problème du Big Data, ce n'est pas seulement de STOCKER des téraoctets ; c'est de pouvoir y CHERCHER, instantanément. Et c'est exactement le problème que résout l'outil dont nous allons parler : Elasticsearch, et sa version open source OpenSearch.

*Notes : ton posé mais qui capte. Pause après « trois heures du matin ». Insister sur « instantanément ». Regarder l'examinateur, pas tes notes.*

## ② Qu'est-ce qu'Elasticsearch + lien OpenSearch (65s)
> Elasticsearch est né en 2010, créé par Shay Banon. L'histoire est presque touchante : il voulait juste un moteur de recherche pour les recettes de cuisine de sa femme. Quinze ans plus tard, environ un milliard et demi de téléchargements. C'est devenu LA référence pour la recherche de texte sur de gros volumes et la surveillance des erreurs en temps réel. Techniquement : codé en Java, s'appuie sur la bibliothèque Apache Lucene qui fait le vrai travail en dessous. On communique avec lui via une API REST, en JSON, et il stocke ses données en documents JSON, pas en tables. Point important pour notre démo : en 2021, Elastic a changé sa licence pour la rendre moins libre. En réaction, Amazon a créé un FORK entièrement open source, OpenSearch. Même ADN, même technologie Lucene, mêmes concepts. Tout ce qu'on dit sur Elasticsearch s'applique à OpenSearch, et notre démo tourne justement sur OpenSearch.

*Notes : l'anecdote des recettes fait sourire. CRUCIAL — poser le lien ES/OpenSearch ICI pour désamorcer la question avant qu'elle arrive. Dire « fork » et « même ADN ».*

## ③ Architecture + le secret de la vitesse : l'index inversé (95s)
> Comment ça marche, et pourquoi c'est si rapide ? Trois idées. Première, le passage à l'échelle : Elasticsearch ne tourne pas sur une machine mais sur un cluster, un groupe de serveurs appelés nœuds. Les données sont rangées dans des index, et chaque index est découpé en morceaux, les shards, répartis sur les nœuds. Si la charge monte, on ajoute des nœuds : c'est l'horizontalité du Big Data. Deuxième, la résilience : chaque shard est copié en répliques sur d'autres nœuds. Si un serveur tombe, une copie prend le relais. Troisième, LE secret : l'index inversé. Au moment où il reçoit les données, Elasticsearch construit à l'avance un dictionnaire qui, pour chaque mot, liste TOUS les documents où ce mot apparaît. Comme l'index à la fin d'un livre : vous ne relisez pas les 500 pages, vous allez à l'index et il vous donne les numéros de page. Voilà pourquoi on cherche dans des millions de documents en quelques millisecondes : le travail dur est fait en amont, à l'écriture, pas à la lecture.

*Notes : le cœur technique. Ralentir sur l'index inversé, la métaphore du livre est ton arme. Geste : feuilleter des pages puis pointer l'index.*

## ④ Positionnement Big Data — le voyage de la donnée (95s)
> Prenons de la hauteur. Elasticsearch ne vit pas seul, il s'insère dans une chaîne. Suivons une donnée. Tout commence par l'ingestion. *(⚠️ pour des logs : Logstash/Beats/Kafka — Sqoop sert seulement à importer depuis des bases SQL).* Ensuite le stockage et le traitement de masse : HDFS garde des volumes énormes à bas coût, Spark et Hive font les gros calculs en batch sur l'historique. À côté, on choisit la base NoSQL adaptée : MongoDB pour des documents souples, Redis pour une réponse en mémoire ultra-rapide, Neo4j quand la donnée est faite de relations. Et tout au bout, là où un humain a besoin de CHERCHER et de VOIR, on place Elasticsearch/OpenSearch : la couche de recherche, la donnée chaude. Message clé : ces outils ne sont PAS concurrents, ils sont COMPLÉMENTAIRES. Hadoop n'a jamais cherché à faire du temps réel, ES n'a jamais cherché à remplacer un data lake. Spark traite, Hadoop conserve, Elasticsearch donne à voir. Chacun son métier.

*Notes : LA section qui fait la différence. Raconter une histoire, pas réciter. Marteler « complémentaires, pas concurrents ». Un verbe par outil : aspire, conserve, calcule, cherche, montre.*

## ⑤ Comparaison NoSQL (40s)
> Pourquoi ne pas mettre nos logs dans MongoDB, Cassandra ou Neo4j directement ? MongoDB stocke très bien des documents, mais la recherche en texte libre n'est pas son cœur de métier. Cassandra est imbattable pour écrire vite à grande échelle, mais elle oblige à savoir d'avance comment on va interroger — impossible d'inventer une recherche libre après coup. Neo4j brille uniquement sur les relations, les graphes, ce qui n'a pas de sens pour des logs. Elasticsearch, lui, est né pour ça : indexer du texte et le restituer instantanément. On ne choisit pas le meilleur outil dans l'absolu, on choisit le bon outil pour le bon besoin.

*Notes : rythme rapide. Une phrase = un outil = une raison. Clôturer sur « le bon outil pour le bon besoin ».*

## ⑥ Transition démo (35s)
> Assez de théorie, passons à la pratique. On travaille sur OpenSearch avec un jeu de logs web réel, ~14 000 documents. Quatre recherches qui illustrent les deux familles de requêtes. D'abord deux requêtes structurées : les erreurs 404, puis un filtre sur une plage de dates. Ensuite deux requêtes plein texte : le mot « error » dans les tags, puis « opensearch » dans le message et l'URL. Et pour finir, un tableau de bord de trafic web. Place à la démonstration.

*Notes : énergie qui remonte. Annoncer les 4 requêtes + la viz pour que le binôme enchaîne. Terminer net.*

---

# PARTIE 2 — LES ENJEUX (le « pourquoi » d'architecte)

**Résumé :** pourquoi combiner Hadoop/Spark d'un côté et OpenSearch de l'autre pour les logs ? Parce que c'est un arbitrage entre coût et vitesse, qu'on résout par le chaud/froid.

1. **Données chaudes vs froides** — la valeur d'un log décroît avec le temps. Les logs récents (chauds) vont dans OpenSearch pour répondre en ms ; les anciens (froids) descendent dans HDFS. OpenSearch gère ça via ses tiers et la rotation d'index (plugin ISM).

2. **Coût de stockage vs vitesse de recherche** — HDFS ne coûte presque rien mais chercher = un job Spark qui scanne tout. L'index inversé Lucene est instantané mais pèse lourd (RAM/SSD chers). D'où : indexer peu (le chaud), archiver beaucoup (le froid).

3. **Observabilité / monitoring temps réel** — Hadoop/Spark = batch (ce qui s'est passé hier). Surveiller une prod exige du near-real-time : OpenSearch ingère en continu et restitue via Dashboards.

4. **Détection d'incidents en secondes** — un batch nocturne arrive trop tard. OpenSearch déclenche une alerte dès qu'un seuil tombe (ex. > 800 codes 404 en 5 min) → socle de SIEM.

5. **Conformité / rétention longue** — RGPD impose des logs gardés des années, immuables. Tout garder dans OpenSearch serait ruineux. Le data lake est la source de vérité légale ; OpenSearch est une vue reconstruisible qu'on réindexe depuis le lac.

---

# PARTIE 3 — Q/R PIÈGES (réponses vérifiées)

## A. Théorie & technique

**Q1 — Pourquoi c'est rapide ? Index inversé vs B-tree SQL ?**
Un index inversé va du **mot → liste des documents** qui le contiennent (posting list). À l'indexation, l'analyzer découpe le texte en tokens ; pour chaque terme on a une liste triée d'IDs de docs. Chercher « error » = localiser le terme + lire sa liste déjà prête, jamais scanner les 14 000 docs. Un B-tree SQL est fait pour des valeurs exactes/plages (`WHERE id=X`) ; pour chercher un mot **au milieu** d'un texte, SQL fait `LIKE '%error%'` → scan complet O(n). En plus ES calcule un **score de pertinence (BM25)**, ce que SQL ne fait pas. La rapidité = on a pré-payé à l'écriture pour rendre la lecture quasi immédiate.

**Q2 — Rôle de l'analyzer/tokenizer ? Cas où ça donne des résultats faux ?**
L'analyzer = pipeline appliqué à l'indexation ET à la requête : character filters → tokenizer (découpe) → token filters (minuscule, stop-words, stemming). Ce qui est stocké dans l'index, ce sont les **tokens analysés**. Donc `match` (full-text) passe par l'analyzer, `term` fait une correspondance **exacte sans ré-analyse**. ⚠️ Un `term` sur « Error » échoue si l'index a tout mis en minuscule. Mauvaise config : mapping en `keyword` au lieu de `text` → `match opensearch` ne trouve rien car le champ n'est pas découpé en mots.

**Q3 — « Segments Lucene immuables » : comment update/delete alors ?**
Un shard = plusieurs segments (mini-index inversés). Une fois écrit, un segment n'est **jamais** modifié. Donc : un nouveau doc part dans un **nouveau** segment ; un *update* = soft-delete de l'ancien + réindexation d'une nouvelle version ; un *delete* = marque le doc supprimé (récupéré seulement au **merge**). Pourquoi l'immutabilité ? Lecture sans verrou, cache agressif, écriture append-only (pas de corruption). Prix : amplification d'écriture (merges) → mauvais pour les updates très fréquents.

**Q4 — Vraies limites d'ES ? On remplace une base transactionnelle ?**
Non. (1) Pas une source de vérité — on garde la donnée maîtresse dans une base ACID et on indexe une **copie**. (2) Pas d'ACID multi-documents (juste concurrence optimiste par doc via `_seq_no`/`_primary_term`). (3) Near-real-time, pas temps réel (refresh ~1s). (4) Coût heap JVM (≤ ~32 Go, compressed oops). (5) Jointures faibles (nested, parent/child coûteux). (6) Re-mapping coûteux (réindexation). → Complément d'une base de vérité, jamais remplaçant.

**Q5 — Refresh vs flush vs commit ? Durabilité si crash ?**
**Refresh** = crée un nouveau segment cherchable (page cache), rend les docs **visibles** (~1s) → c'est le near-real-time. La **durabilité** ne dépend PAS du refresh : chaque écriture est consignée dans le **translog** (write-ahead log), fsync sur disque (mode `request` par défaut). Si crash, le translog est rejoué au redémarrage. **Flush** = commit Lucene : écrit durablement les segments (fsync) et vide le translog. → refresh = cherchable, flush = durable.

**Q6 — Nœud master vs data ? Autres rôles ?**
**Master** : gère l'ÉTAT du cluster (création d'index, allocation des shards, suivi des nœuds), PAS la donnée. Un seul master élu actif. **Data** : stocke les shards, exécute indexation/recherche/agrégations. **Ingest** : pipelines de pré-traitement. **Coordinating** : reçoit la requête, route vers les shards, agrège les résultats. ⚠️ Erreur classique : croire que le master traite les données.

**Q7 — Split-brain ? Avant/après ES 7 ?**
Partition réseau → chaque groupe élit son propre master → deux masters qui acceptent des écritures → divergence et perte de données. Solution = **quorum** : élection autorisée seulement si majorité de nœuds master-eligible : (N/2)+1.
- **Avant ES 7** : réglé MANUELLEMENT via `discovery.zen.minimum_master_nodes` = (N/2)+1. Source d'erreurs humaines.
- **ES 7+ / OpenSearch** : nouveau module de coordination, paramètre manuel SUPPRIMÉ, quorum géré automatiquement par une **voting configuration** (on configure juste `cluster.initial_master_nodes` au 1er démarrage). Split-brain évité par construction.

**Q8 — Shard primaire vs réplica ? Pourquoi le nb de primaires est figé ?**
Primaire = partition autonome de l'index (écriture d'abord). Réplica = copie sur un autre nœud (haute dispo + montée en charge en lecture, jamais sur le même nœud que son primaire). Le nb de primaires est figé car le routing = `hash(_id) % nombre_de_shards_primaires`. Changer ce nombre changerait le modulo → on ne retrouverait plus les docs. Les réplicas n'entrent pas dans la formule → modifiables à chaud. Pour vraiment changer : `_split`/`_shrink` ou réindexer via alias.

**Q9 — Un nœud data tombe pendant des recherches : que se passe-t-il ?**
Avec ≥ 1 réplica : le master détecte la panne, **promeut un réplica en primaire**, la recherche continue sans coupure ni perte, le cluster passe **JAUNE** (réplicas manquants), puis ré-alloue de nouveaux réplicas → **VERT**. Sans réplica (0) : shards perdus → cluster **ROUGE**, recherches partielles, perte de données. → En prod : toujours ≥ 1 réplica + allocation awareness.

---

## B. Positionnement vs NoSQL

**Q1 — Pourquoi pas tout dans MongoDB (qui a un index full-text) ?**
Vrai, MongoDB a un index `$text`, donc « il ne sait pas chercher » serait faux. La vraie différence : MongoDB est d'abord orienté **stockage/CRUD**, son full-text est limité (pas de scoring riche, pas d'analyzers fins par langue, un seul index text par collection). ES/OpenSearch est **nativement un moteur de recherche** Lucene : scoring BM25, analyzers fins, et surtout **agrégations analytiques** en near-real-time. ⚠️ Nuance : MongoDB **Atlas Search** est basé sur Lucene et comble une partie de l'écart — mais c'est un service Atlas, pas le moteur de base.

**Q2 — Pourquoi pas Cassandra (imbattable en écriture) ?**
Cassandra = excellente pour l'ingestion (colonnes larges, écritures massives distribuées, time-series haut débit). Mais modèle **query-first** : il faut modéliser selon les requêtes connues d'avance. Recherche ad-hoc / full-text / agrégation flexible → pauvre. ⚠️ Elle a des index secondaires (SASI, SAI depuis v5.0) mais limités ; le vrai full-text passait par DSE Search (Solr/Lucene). Souvent on combine : Cassandra/Kafka pour le flux + ES pour la recherche.

**Q3 — Pourquoi pas Neo4j ?**
Neo4j n'est pas « moins bon », il répond à un autre problème : les **relations** et leur parcours (fraude, réseaux sociaux, recommandation). Des logs web sont des documents plats et indépendants, pas de question « plus court chemin » ou « voisinage à N sauts ». Le critère = la forme de la donnée.

**Q4 — Pourquoi pas Redis (ultra-rapide) ?**
Redis = clé-valeur en RAM, imbattable pour récupérer une valeur **quand on connaît la clé** (cache, sessions, compteurs). Mais il **n'indexe pas le contenu** : « tous les logs contenant error en mai avec status 404 » → impossible sans avoir prévu la clé. ⚠️ Le module RediSearch ajoute du full-text, mais ce n'est pas le cœur de Redis. Redis serait plutôt un cache **devant** ES.

**Q5 — OpenSearch est-il une base NoSQL ou un moteur de recherche ?**
Les deux, avec une hiérarchie. Techniquement : traits NoSQL documentaire (stocke du JSON, API REST). Fonctionnellement : avant tout un **moteur de recherche/analytics** (index inversé, BM25, agrégations). Conséquence : on ne l'utilise quasiment jamais comme **base primaire** (pas d'ACID multi-docs, near-real-time, re-mapping lourd). → « magasin de données orienté recherche », déployé **au-dessus** d'une vraie base.

**Q6 — On remplace MongoDB/PostgreSQL par ES alors ?**
Non, erreur d'architecture. La base reste la **source de vérité** (durabilité, cohérence forte, ACID). ES vient **par-dessus**, alimenté par cette base, pour la recherche. Si on perd l'index ES → on le reconstruit depuis la source. Pattern : « base = vérité, ES = vue de recherche dérivée ».

**Q7 — Nature exacte de chaque base en une phrase ?**
MongoDB = documentaire JSON (stockage/CRUD souple). Cassandra = colonnes larges (écriture massive, time-series). Neo4j = graphe (relations). Redis = clé-valeur RAM (cache/accès par clé). OpenSearch = moteur de recherche/analytics sur index inversé (full-text + agrégations). **Critère unique** : forme de la donnée × type de requête dominant.

**Q8 — Démo OpenSearch mais slides Elasticsearch : ça change la compa NoSQL ?**
Non, rien. OpenSearch = fork d'ES (AWS, 2021), même base Lucene/Query DSL/agrégations → même nature, même raisonnement de choix. La différence est licence/gouvernance, qui se joue **contre Elasticsearch**, pas contre les autres NoSQL.

---

## C. Architecture Big Data

**Q1 — Pourquoi pas tout dans Hadoop/Hive ?**
Pas le même besoin. Hadoop/Hive = **batch** : requête sur des To d'historique, réponse en minutes/heures, modèle full-scan. ES = **near-real-time** : index inversé, recherche + agrégations en ms/s. Pendant un incident, un ops cherchant les 404 des 5 dernières min ne peut pas attendre un job MapReduce. → On combine, on ne remplace pas.

**Q2 — Sqoop ingère vos 14 000 logs dans OpenSearch ? (PIÈGE)**
⚠️ **NON.** Sqoop = transfert batch entre base **relationnelle** (JDBC) et Hadoop/HDFS, besoin d'un schéma SQL. Des logs = flux continu semi-structuré, pas d'une base SQL. Pour ingérer des logs → **Filebeat/Fluentd**, **Logstash** (parsing), **Kafka** (tampon). Sur la slide, Sqoop illustre l'ingestion depuis des bases relationnelles, pas le pipeline de logs.

**Q3 — Pipeline complet d'un log, du serveur au dashboard ?**
Serveur web écrit ses logs → **Filebeat** lit → **Kafka** (tampon/découplage). Chemin chaud : **Logstash / Spark Streaming** parse + enrichit (géoloc IP, user-agent) → indexe dans **OpenSearch** → dashboard. Chemin froid : logs bruts → **HDFS** (archivage), Spark/Hive pour l'historique. Brique de trop ici = **Sqoop** (pas de base relationnelle source).

**Q4 — Batch / streaming / near-real-time : définitions + placement ?**
**Batch** : gros lot en différé, optimise le débit (Hadoop/MapReduce, Hive, Spark batch). **Streaming** : au fil de l'eau (Kafka transport, Spark Structured Streaming/Flink). **Near-real-time** : disponible très vite après arrivée, pas instantané → ES/OpenSearch (refresh ~1s). ⚠️ ES n'est PAS du temps réel strict. ⚠️ Spark Streaming = **micro-batch** ; Flink = événement par événement.

**Q5 — Stack ELK et équivalents OpenSearch ?**
ELK = **E**lasticsearch (stockage/recherche) + **L**ogstash (ingestion/transfo) + **K**ibana (visu), + Beats en amont. Équivalents OpenSearch : Elasticsearch → **OpenSearch**, Kibana → **OpenSearch Dashboards**, Logstash → **Data Prepper** (et Logstash via output plugin). ⚠️ « OpenSearch Beats » n'existe pas sous ce nom : collecte via **Fluent Bit / Fluentd / OpenTelemetry**.

**Q6 — ELK = architecture Lambda ?**
Non. ELK = pipeline mono-chemin (ingestion/indexation/visu). **Lambda** = batch layer (Hadoop/Spark, vues exactes, latence) + speed layer (Kafka+Spark/Flink, temps réel) + serving layer (← **ES** s'insère ici). Beaucoup vont vers **Kappa** (tout-streaming via Kafka) pour éviter de maintenir deux pipelines.

**Q7 — Pourquoi avoir Hadoop ET ES ? Tout dans ES ?**
Arbitrage **vitesse vs coût** → chaud/froid. ES = rapide mais cher (RAM/SSD, index inversé, réplicas). HDFS = disque banal, bas coût/To, mais lent. Chaud (jours/semaines) dans OpenSearch ; froid (historique) dans HDFS. Chacun là où il est le meilleur.

**Q8 — ES = une base de données ? Base principale ?**
Ressemble à une base documentaire (JSON, scale, agrégations) mais **pas un système de vérité**. Pas d'ACID multi-docs, optimisé lecture/recherche (pas writes transactionnels), mapping pénible à faire évoluer. → Source de vérité dans une vraie base, ES = **index secondaire de recherche** alimenté par elle. On peut reconstruire l'index depuis la source, pas l'inverse.

---

## D. Licences & coûts

**Q1 — Elasticsearch est-il vraiment open source ?**
Ça dépend de l'année. **2010–2021** : Apache 2.0 (vrai open source OSI). **2021 (v7.11)** : abandon d'Apache 2.0 → double licence **SSPL + Elastic License v2** (ni l'une ni l'autre OSI → « source-available »). **2024 (v8.16)** : ajout de l'**AGPL v3** (approuvée OSI) → de nouveau open source via cette option. → Vrai (2010-21), faux (21-24), de nouveau vrai (depuis 24). OpenSearch, lui, est resté **Apache 2.0** sans interruption.

**Q2 — Différence ES / OpenSearch et pourquoi le fork ?**
Conflit Elastic vs AWS (AWS vendait du managé Elasticsearch). Janvier 2021 : Elastic change de licence pour bloquer ça. AWS forke la dernière version Apache 2.0 (**Elasticsearch 7.10.2**) → crée **OpenSearch** (+ OpenSearch Dashboards, fork de Kibana) sous Apache 2.0. Base commune (Lucene, index inversé, Query DSL), divergence depuis 2021. ⚠️ Aussi un litige sur la **marque** « Elasticsearch ».

**Q3 — SSPL vs Elastic License v2 vs Apache 2.0 vs AGPL ?**
- **Apache 2.0** : permissive, OSI, quasi aucune contrainte (← OpenSearch).
- **SSPL** (créée par MongoDB) : copyleft agressif — offrir en SaaS oblige à publier toute la stack de service → **refusée par l'OSI**.
- **Elastic License v2** : source-available, quasi propriétaire (pas de service managé concurrent).
- **AGPL v3** : libre, OSI, copyleft « réseau » mais obligations jugées raisonnables → open source.
→ Apache 2.0 & AGPL = open source ; SSPL & ELv2 = source-available.

**Q4 — Si Elastic a remis l'AGPL en 2024, le fork ne sert plus à rien ?**
Non. (1) **Confiance** : Elastic a changé unilatéralement, peut recommencer. (2) **Gouvernance** : OpenSearch est sous la **Linux Foundation** (2024), Elasticsearch reste piloté par une seule entreprise. (3) **Écosystème** : divergence depuis 2021, base d'utilisateurs propre. Le retour de l'AGPL est justement une réaction stratégique parce qu'OpenSearch leur prenait des parts.

**Q5 — Apache 2.0 + gratuit = déployer ne coûte rien ?**
Le **logiciel** est gratuit, l'**exploitation** non. Coût = infra (RAM, stockage), réplicas (doublent le disque), CPU (agrégations, scoring), et surtout **humain** (dimensionnement, monitoring, montées de version). C'est ce modèle qui explique le conflit de licences : moteur gratuit, revenus sur le managé/support.

**Q6 — Combien de RAM ? Limite ?**
(1) Allouer ~**50 % de la RAM** au heap JVM, l'autre moitié à l'OS (Lucene s'appuie sur le **filesystem cache**). (2) Plafond ~**31-32 Go de heap** : sous ce seuil la JVM utilise les **compressed oops** (pointeurs 32 bits) ; au-delà, pointeurs 64 bits → 40 Go peut donner moins de mémoire utile que 31. → plusieurs nœuds à ~31 Go plutôt qu'un seul à 64.

**Q7 — Où vont les coûts à l'échelle ?**
**Stockage** : index inversé + **doc_values** (colonne, tris/agrégations) + `_source` → plus que les données brutes ; + **réplicas** (× au moins 2). **Calcul** : agrégations/tri (CPU + mémoire, surtout forte cardinalité). Parade : **hot-warm-cold-frozen** (récent sur SSD chers, ancien vers stockage moins cher / S3), baisser les réplicas sur les vieux index.

**Q8 — Risque le plus coûteux en prod ?**
Le **mauvais dimensionnement des shards**. Trop de shards (over-sharding) → surcoût heap/métadonnées. Trop peu → pas de parallélisme. Et on ne peut pas changer le nb de primaires sans réindexer. Sous-dimensionné → GC/nœuds qui tombent ; sur-dimensionné → facture cloud gaspillée. Le vrai coût = une mauvaise décision d'archi qu'on traîne longtemps.

---

## E. Démo & optimisation

**Q1 — En interne, term vs match : que se passe-t-il ? Le score ?**
`term response=404` = question binaire (correspond ou non). En **filter context** (sous bool/filter) → **aucun score**, résultat **cacheable** (node query cache). `match tags=error` = **query context** : passe par l'analyzer, calcule un **_score BM25** (fréquence du terme, rareté via IDF, longueur du champ), tri par pertinence. ⚠️ Honnêteté : au top-level mon `term` renvoie un score constant 1.0 ; pour profiter du cache il faut l'envelopper dans `bool/filter`.

**Q2 — Pourquoi un `term` sur un champ texte renvoie 0 résultat ?**
`term` fait une comparaison **exacte sans analyzer**. Un champ `text` est analysé à l'indexation (minuscules + tokenisation) → « Error » stocké comme token « error ». `term: { tags: "Error" }` cherche littéralement « Error » → **0 résultat**, alors que `match` fonctionnerait (il ré-analyse). D'où le sous-champ **`.keyword`** (non analysé) pour du term exact sur du texte. Règle : **term sur keyword/numérique, match sur text**.

**Q3 — `response=404` : term ou match ? Et term sur `message` ?**
Sur `response` (valeur exacte, code HTTP) → **term** : on veut un filtre oui/non, pas un score. `match` marcherait mais part inutilement en query context. ⚠️ Sur `message` (text analysé) → **term serait une erreur**, il faut `match`. Le bon opérateur dépend du **type/mapping du champ**, pas de l'habitude.

**Q4 — multi_match « opensearch » = 5101/14000, pertinent ou bruit ?**
Pas faux mais trompeur. Le dataset simule du trafic vers les artefacts OpenSearch → « opensearch » est partout (URLs, messages). Le champ `url` est analysé → découpé en tokens, « opensearch » apparaît plusieurs fois → gonfle les matches. Mais contrairement à term/range, le multi_match **trie par _score BM25** : les plus pertinents remontent. Au client : « 5101 documents contiennent le terme, les plus pertinents sont en haut ». Réduire le bruit : top-N, `minimum_score`, boost `message^2`, ou `match_phrase`.

**Q5 — filter context + cache : où placer quoi en prod ?**
Sur des logs, les contraintes répétitives (range timestamp, term response, service) → **filter context** (clause `filter` d'un bool) : pas de score + résultat caché en **bitset** (node query cache). Un dashboard qui rejoue « dernières 24h » 200× réutilise le bitset → quasi gratuit. Le **query context** (`must` avec match) ne reçoit QUE la partie où la pertinence compte. Pattern : `bool: { must: [match message], filter: [range timestamp, term response] }` → on ne paie le BM25 que sur le sous-ensemble déjà réduit.

**Q6 — Agrégation `terms` : coût ? pourquoi `size:0` ? danger ?**
`terms` lit les **doc_values** (colonne, ⚠️ sur **disque/off-heap**, pas le heap). `size: 0` = « pas de docs de détail, juste l'agrégation » → économise réseau/CPU. **Danger = la cardinalité** : `terms` sur `response` (5-6 valeurs) trivial ; sur `ip` ou un ID unique (forte cardinalité) → millions de buckets, risque de saturation + **doc_count_error** sur les top-N distribués. Parade : borner `size`, agg `cardinality` (HyperLogLog, approx), ou `composite` pour paginer.

**Q7 — Comment ES répond en ms sur 14 000 docs ? Et à 14 millions ?**
À 14 000, presque tout serait rapide — la vraie question = le **scaling**. (1) **Index inversé** : posting list pré-calculée, coût ∝ nb de termes cherchés, pas taille du dataset. (2) **Sharding** : index découpé en shards sur plusieurs nœuds, requête **parallèle**, fusion par le coordinateur. (3) **Réplicas** : répartition de charge en lecture. + filter cache + doc_values. ⚠️ Reste rapide **si le sharding est bien dimensionné** — le scaling se conçoit, il n'est pas gratuit.

**Q8 — Requêtes identiques sur ES ? Piège scoring/licence ?**
Oui. OpenSearch part d'**Elasticsearch 7.10.2** (Apache 2.0) → même **Query DSL** (term, range, match, multi_match, bool, agrégations), même **BM25** par défaut. Pièges : (1) projets qui **divergent** depuis 2021 (versions, plugins, ML/sécurité diffèrent) → pas 100 % compatibles sur les features récentes ; (2) compat annoncée avec ES **7.10**, pas avec les 8.x. → ES = concept/origine, OpenSearch = implémentation libre, même moteur Lucene.

---

*Fin — bonne chance pour l'oral.*
