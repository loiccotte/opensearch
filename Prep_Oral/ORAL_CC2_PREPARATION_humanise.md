# Préparation oral CC2 : Elasticsearch / OpenSearch

> 7 min de présentation + 3 min de Q/R, examinateur strict, en binôme.
> Ce document, c'est mon aide de prépa : le pitch à dire, les enjeux, et les questions pièges avec leurs réponses.

---

## Les 5 corrections critiques (à connaître absolument)

Le prof va taper exactement là-dessus. Ce sont des erreurs qui traînent dans les slides et le discours par défaut.

1. **Sqoop n'est pas un outil d'ingestion de logs.** Sqoop, c'est du transfert batch entre SQL et Hadoop (JDBC, données structurées). C'est même un projet retiré, parti à l'Apache Attic. Pour ingérer des logs, on passe par Logstash, Beats, Fluentd, Data Prepper côté OpenSearch, ou Kafka. Et Sqoop est sur ma slide 2.1, donc le piège est quasiment garanti.

2. **Les licences en 2026.** Dire "OpenSearch est libre, Elasticsearch ne l'est plus", c'est périmé. Depuis août 2024 (la v8.16, sous licence AGPLv3), Elasticsearch est redevenu open source. Aujourd'hui la vraie différence est surtout une question de gouvernance : OpenSearch est en Apache 2.0, sous la Linux Foundation depuis 2024.

3. **term contre match, c'est "non-analysé contre analysé"**, pas "structuré contre full-text". Un `term` ne passe jamais par l'analyzer. Du coup un `term` sur un champ `text` peut renvoyer zéro résultat à cause de la casse, parce que le token stocké est en minuscule.

4. **Les agrégations utilisent les `doc_values`**, c'est-à-dire du stockage en colonne, sur disque ou off-heap. Elles ne passent pas par l'index inversé. Et les doc_values ne consomment pas le heap, contrairement à l'ancien fielddata.

5. **Index inversé contre SQL.** Il ne faut jamais dire "SQL parcourt tout ligne par ligne". SQL a des index B-tree. La vraie différence, c'est le full-text : chercher un mot au milieu d'un texte. C'est un `LIKE '%mot%'` en SQL, et celui-là, oui, il scanne tout.

---

# Partie 1 : le pitch (7 min, environ 420s)

## 1. Accroche (45s)

> Imaginez. Il est trois heures du matin, votre site e-commerce ne répond plus, et quelque part dans vos serveurs, plusieurs millions de lignes de logs cachent LA ligne qui explique la panne. Avec une base de données classique, retrouver cette ligne, c'est chercher une aiguille dans une botte de foin : plusieurs minutes, parfois des heures. Or quand un site est à terre, chaque minute coûte de l'argent. Le vrai problème du Big Data, ce n'est pas seulement de STOCKER des téraoctets, c'est de pouvoir y CHERCHER, instantanément. Et c'est exactement le problème que résout l'outil dont on va parler : Elasticsearch, et sa version open source OpenSearch.

*Notes : ton posé mais qui capte. Pause après "trois heures du matin". Insister sur "instantanément". Regarder l'examinateur, pas mes notes.*

## 2. Qu'est-ce qu'Elasticsearch, et le lien avec OpenSearch (65s)

> Elasticsearch est né en 2010, créé par Shay Banon. L'histoire est presque touchante : il voulait juste un moteur de recherche pour les recettes de cuisine de sa femme. Quinze ans plus tard, on en est à environ un milliard et demi de téléchargements. C'est devenu LA référence pour la recherche de texte sur de gros volumes et la surveillance des erreurs en temps réel. Côté technique : c'est codé en Java, et ça s'appuie sur la bibliothèque Apache Lucene, qui fait le vrai travail en dessous. On lui parle via une API REST, en JSON, et il stocke ses données en documents JSON, pas en tables. Un point important pour notre démo : en 2021, Elastic a changé sa licence pour la rendre moins libre. En réaction, Amazon a créé un fork entièrement open source, OpenSearch. Même ADN, même techno Lucene, mêmes concepts. Tout ce qu'on dit sur Elasticsearch s'applique à OpenSearch, et notre démo tourne justement sur OpenSearch.

*Notes : l'anecdote des recettes fait sourire. Important, poser le lien ES/OpenSearch ICI pour désamorcer la question avant qu'elle arrive. Dire "fork" et "même ADN".*

## 3. L'architecture, et le secret de la vitesse : l'index inversé (95s)

> Comment ça marche, et pourquoi c'est si rapide ? Trois idées. D'abord le passage à l'échelle. Elasticsearch ne tourne pas sur une machine mais sur un cluster, un groupe de serveurs qu'on appelle des nœuds. Les données sont rangées dans des index, et chaque index est découpé en morceaux, les shards, répartis sur les nœuds. Si la charge monte, on ajoute des nœuds : c'est l'horizontalité du Big Data. Ensuite la résilience. Chaque shard est copié en répliques sur d'autres nœuds, donc si un serveur tombe, une copie prend le relais. Et enfin LE secret : l'index inversé. Au moment où il reçoit les données, Elasticsearch construit à l'avance un dictionnaire qui, pour chaque mot, liste TOUS les documents où ce mot apparaît. Comme l'index à la fin d'un livre : vous ne relisez pas les 500 pages, vous allez à l'index et il vous donne les numéros de page. Voilà pourquoi on cherche dans des millions de documents en quelques millisecondes : le travail dur est fait en amont, à l'écriture, pas à la lecture.

*Notes : le cœur technique. Ralentir sur l'index inversé, la métaphore du livre est mon arme. Geste : feuilleter des pages puis pointer l'index.*

## 4. Le positionnement Big Data : le voyage de la donnée (95s)

> Prenons de la hauteur. Elasticsearch ne vit pas seul, il fait partie d'une chaîne. Suivons une donnée. Tout commence par l'ingestion. *(Attention, pour des logs : Logstash, Beats ou Kafka. Sqoop sert seulement à importer depuis des bases SQL.)* Ensuite vient le stockage et le traitement de masse : HDFS garde des volumes énormes à bas coût, Spark et Hive font les gros calculs en batch sur l'historique. À côté, on choisit la base NoSQL adaptée : MongoDB pour des documents souples, Redis pour une réponse en mémoire ultra-rapide, Neo4j quand la donnée est faite de relations. Et tout au bout, là où un humain a besoin de CHERCHER et de VOIR, on place Elasticsearch ou OpenSearch : la couche de recherche, la donnée chaude. Le message clé, c'est que ces outils ne sont PAS concurrents, ils sont COMPLÉMENTAIRES. Hadoop n'a jamais cherché à faire du temps réel, ES n'a jamais cherché à remplacer un data lake. Spark traite, Hadoop conserve, Elasticsearch donne à voir. Chacun son métier.

*Notes : la section qui fait la différence. Raconter une histoire, pas réciter. Marteler "complémentaires, pas concurrents". Un verbe par outil : aspire, conserve, calcule, cherche, montre.*

## 5. Comparaison avec les bases NoSQL (40s)

> Pourquoi ne pas mettre nos logs directement dans MongoDB, Cassandra ou Neo4j ? MongoDB stocke très bien des documents, mais la recherche en texte libre n'est pas son cœur de métier. Cassandra est imbattable pour écrire vite à grande échelle, sauf qu'elle oblige à savoir d'avance comment on va l'interroger : impossible d'inventer une recherche libre après coup. Neo4j brille uniquement sur les relations, les graphes, ce qui n'a aucun sens pour des logs. Elasticsearch, lui, est né pour ça : indexer du texte et le restituer instantanément. On ne choisit pas le meilleur outil dans l'absolu, on choisit le bon outil pour le bon besoin.

*Notes : rythme rapide. Une phrase, un outil, une raison. Clôturer sur "le bon outil pour le bon besoin".*

## 6. Transition vers la démo (35s)

> Assez de théorie, passons à la pratique. On travaille sur OpenSearch avec un jeu de logs web réel, environ 14 000 documents. Quatre recherches qui illustrent les deux familles de requêtes. D'abord deux requêtes structurées : les erreurs 404, puis un filtre sur une plage de dates. Ensuite deux requêtes plein texte : le mot "error" dans les tags, puis "opensearch" dans le message et l'URL. Et pour finir, un tableau de bord de trafic web. Place à la démonstration.

*Notes : l'énergie qui remonte. Annoncer les 4 requêtes et la viz pour que le binôme enchaîne. Terminer net.*

---

# Partie 2 : les enjeux (le "pourquoi" d'architecte)

**En résumé :** pourquoi combiner Hadoop/Spark d'un côté et OpenSearch de l'autre pour les logs ? Parce que c'est un arbitrage entre coût et vitesse, et qu'on le résout par le chaud/froid.

1. **Données chaudes contre froides.** La valeur d'un log décroît avec le temps. Les logs récents, les chauds, vont dans OpenSearch pour répondre en millisecondes. Les anciens, les froids, descendent dans HDFS. OpenSearch gère ça avec ses tiers et la rotation d'index (le plugin ISM).

2. **Coût de stockage contre vitesse de recherche.** HDFS ne coûte presque rien, mais chercher dedans, c'est un job Spark qui scanne tout. L'index inversé de Lucene est instantané mais pèse lourd, et la RAM comme le SSD coûtent cher. D'où la logique : indexer peu (le chaud), archiver beaucoup (le froid).

3. **Observabilité et monitoring temps réel.** Hadoop et Spark, c'est du batch, donc ce qui s'est passé hier. Surveiller une prod demande du near-real-time : OpenSearch ingère en continu et restitue via les Dashboards.

4. **Détection d'incidents en quelques secondes.** Un batch nocturne arrive trop tard. OpenSearch déclenche une alerte dès qu'un seuil tombe, par exemple plus de 800 codes 404 en 5 minutes. C'est le socle d'un SIEM.

5. **Conformité et rétention longue.** Le RGPD impose de garder des logs pendant des années, immuables. Tout garder dans OpenSearch serait ruineux. Du coup le data lake est la source de vérité légale, et OpenSearch n'est qu'une vue reconstruisible qu'on réindexe depuis le lac.

---

# Partie 3 : Q/R pièges (réponses vérifiées)

## A. Théorie et technique

**Q1. Pourquoi c'est rapide ? Index inversé contre B-tree SQL ?**

Un index inversé va du mot vers la liste des documents qui le contiennent, ce qu'on appelle la posting list. À l'indexation, l'analyzer découpe le texte en tokens, et pour chaque terme on a une liste triée d'IDs de docs. Chercher "error", c'est localiser le terme puis lire sa liste déjà prête, jamais scanner les 14 000 docs. Un B-tree SQL, lui, est fait pour des valeurs exactes ou des plages, du genre `WHERE id=X`. Pour chercher un mot au milieu d'un texte, SQL fait `LIKE '%error%'`, et là c'est un scan complet en O(n). En plus, ES calcule un score de pertinence avec **BM25**, ce que SQL ne fait pas. La rapidité vient de là : on a pré-payé à l'écriture pour rendre la lecture quasi immédiate.

**Q2. Le rôle de l'analyzer et du tokenizer ? Un cas où ça donne des résultats faux ?**

L'analyzer, c'est un pipeline appliqué à l'indexation ET à la requête : character filters, puis tokenizer qui découpe, puis token filters qui passent en minuscule, retirent les stop-words, font le stemming. Ce qui est stocké dans l'index, ce sont les tokens analysés. Donc `match`, en full-text, passe par l'analyzer, alors que `term` fait une correspondance exacte sans ré-analyse. Le piège : un `term` sur "Error" échoue si l'index a tout mis en minuscule. Autre mauvaise config classique, un mapping en `keyword` au lieu de `text` : là un `match opensearch` ne trouve rien, parce que le champ n'est pas découpé en mots.

**Q3. Les "segments Lucene immuables" : alors comment on fait un update ou un delete ?**

Un shard, c'est plusieurs segments, des mini-index inversés. Une fois écrit, un segment n'est jamais modifié. Du coup, un nouveau doc part dans un nouveau segment. Un update, c'est un soft-delete de l'ancien plus la réindexation d'une nouvelle version. Un delete, ça marque juste le doc comme supprimé, et il n'est vraiment récupéré qu'au merge. Pourquoi cette immutabilité ? Ça permet la lecture sans verrou, un cache agressif, et de l'écriture append-only sans corruption. Le prix à payer, c'est l'amplification d'écriture à cause des merges, ce qui est mauvais pour les updates très fréquents.

**Q4. Les vraies limites d'ES ? On remplace une base transactionnelle avec ?**

Non. Déjà, ce n'est pas une source de vérité : on garde la donnée maîtresse dans une base ACID et on indexe juste une copie. Pas d'ACID multi-documents non plus, on a seulement de la concurrence optimiste par doc via `_seq_no` et `_primary_term`. C'est du near-real-time, pas du temps réel, avec un refresh d'environ 1s. Il y a le coût du heap JVM, plafonné autour de 32 Go à cause des compressed oops. Les jointures sont faibles, le nested et le parent/child coûtent cher. Et le re-mapping est coûteux puisqu'il faut réindexer. Bref, c'est le complément d'une base de vérité, jamais son remplaçant.

**Q5. Refresh, flush, commit : quelle différence ? Et la durabilité si ça crashe ?**

Le **refresh** crée un nouveau segment cherchable en page cache, ce qui rend les docs visibles en 1s environ : c'est ça, le near-real-time. Mais attention, la durabilité ne dépend PAS du refresh. Chaque écriture est d'abord consignée dans le translog, un write-ahead log, avec un fsync sur disque (en mode `request` par défaut). Si ça crashe, le translog est rejoué au redémarrage. Le **flush**, c'est le commit Lucene : il écrit durablement les segments avec fsync et vide le translog. Donc, pour résumer : refresh rend cherchable, flush rend durable.

**Q6. Nœud master contre nœud data ? Les autres rôles ?**

Le **master** gère l'ÉTAT du cluster : création d'index, allocation des shards, suivi des nœuds. Pas la donnée. Et il n'y a qu'un seul master élu actif. Le nœud **data** stocke les shards et exécute l'indexation, la recherche, les agrégations. Le nœud **ingest** s'occupe des pipelines de pré-traitement. Le **coordinating** reçoit la requête, la route vers les shards et agrège les résultats. L'erreur classique, c'est de croire que le master traite les données.

**Q7. Le split-brain ? Avant et après ES 7 ?**

Une partition réseau, et chaque groupe élit son propre master. On se retrouve avec deux masters qui acceptent des écritures, donc divergence et perte de données. La solution, c'est le quorum : une élection n'est autorisée que si on a la majorité des nœuds master-eligible, soit (N/2)+1.

- Avant ES 7, ça se réglait à la main avec `discovery.zen.minimum_master_nodes` = (N/2)+1. Belle source d'erreurs humaines.
- Avec ES 7+ et OpenSearch, il y a un nouveau module de coordination. Le paramètre manuel a disparu, le quorum est géré automatiquement par une voting configuration. On configure juste `cluster.initial_master_nodes` au premier démarrage. Le split-brain est évité par construction.

**Q8. Shard primaire contre réplica ? Pourquoi le nombre de primaires est figé ?**

Le primaire, c'est une partition autonome de l'index, où l'écriture arrive d'abord. Le réplica, c'est une copie sur un autre nœud, pour la haute dispo et la montée en charge en lecture, jamais sur le même nœud que son primaire. Le nombre de primaires est figé parce que le routing se fait par `hash(_id) % nombre_de_shards_primaires`. Si on change ce nombre, on change le modulo, et on ne retrouve plus les docs. Les réplicas, eux, n'entrent pas dans la formule, donc on peut les modifier à chaud. Pour vraiment changer le nombre de primaires, il faut passer par `_split`, `_shrink`, ou réindexer via un alias.

**Q9. Un nœud data tombe pendant des recherches : qu'est-ce qui se passe ?**

Avec au moins 1 réplica, le master détecte la panne, promeut un réplica en primaire, et la recherche continue sans coupure ni perte. Le cluster passe en JAUNE, parce qu'il manque des réplicas, puis il en ré-alloue de nouveaux et repasse VERT. Sans réplica (à 0), les shards sont perdus : le cluster passe en ROUGE, les recherches deviennent partielles, et il y a perte de données. Conclusion en prod : toujours au moins 1 réplica, plus de l'allocation awareness.

---

## B. Positionnement face aux NoSQL

**Q1. Pourquoi pas tout dans MongoDB, qui a pourtant un index full-text ?**

C'est vrai, MongoDB a un index `$text`, donc dire "il ne sait pas chercher" serait faux. La vraie différence : MongoDB est d'abord orienté stockage et CRUD, son full-text est limité (pas de scoring fin, pas d'analyzers détaillés par langue, un seul index text par collection). ES et OpenSearch, eux, sont nativement des moteurs de recherche Lucene : scoring BM25, analyzers fins, et surtout des agrégations analytiques en near-real-time. Une nuance honnête quand même : MongoDB Atlas Search est basé sur Lucene et comble une partie de l'écart. Mais c'est un service Atlas, pas le moteur de base.

**Q2. Pourquoi pas Cassandra, imbattable en écriture ?**

Cassandra est excellente pour l'ingestion : colonnes larges, écritures massives distribuées, time-series à haut débit. Mais c'est un modèle query-first : il faut modéliser selon les requêtes qu'on connaît à l'avance. Pour de la recherche ad-hoc, du full-text, de l'agrégation flexible, elle est pauvre. À noter, elle a des index secondaires (SASI, et SAI depuis la v5.0), mais limités. Le vrai full-text passait par DSE Search, en Solr/Lucene. D'ailleurs on combine souvent les deux : Cassandra ou Kafka pour le flux, et ES pour la recherche.

**Q3. Pourquoi pas Neo4j ?**

Neo4j n'est pas "moins bon", il répond simplement à un autre problème : les relations et leur parcours, comme la fraude, les réseaux sociaux, la recommandation. Or des logs web sont des documents plats et indépendants. Pas de question du genre "plus court chemin" ou "voisinage à N sauts". Le critère, c'est la forme de la donnée.

**Q4. Pourquoi pas Redis, qui est ultra-rapide ?**

Redis, c'est du clé-valeur en RAM, imbattable pour récupérer une valeur quand on connaît la clé : cache, sessions, compteurs. Mais il n'indexe pas le contenu. "Tous les logs contenant error en mai avec un status 404", c'est impossible sans avoir prévu la clé à l'avance. Le module RediSearch ajoute du full-text, mais ce n'est pas le cœur de Redis. Redis serait plutôt un cache placé devant ES.

**Q5. OpenSearch, c'est une base NoSQL ou un moteur de recherche ?**

Les deux, mais avec une hiérarchie. Techniquement, il a des traits de NoSQL documentaire : il stocke du JSON, il a une API REST. Mais fonctionnellement, c'est avant tout un moteur de recherche et d'analytics : index inversé, BM25, agrégations. La conséquence, c'est qu'on ne l'utilise quasiment jamais comme base primaire, à cause de l'absence d'ACID multi-docs, du near-real-time, et du re-mapping lourd. Donc je dirais plutôt un "magasin de données orienté recherche", déployé au-dessus d'une vraie base.

**Q6. On remplace MongoDB ou PostgreSQL par ES, alors ?**

Non, ce serait une erreur d'architecture. La base reste la source de vérité : durabilité, cohérence forte, ACID. ES vient par-dessus, alimenté par cette base, juste pour la recherche. Si on perd l'index ES, on le reconstruit depuis la source. Le pattern à retenir : la base est la vérité, ES est une vue de recherche dérivée.

**Q7. La nature exacte de chaque base, en une phrase ?**

MongoDB, c'est du documentaire JSON, pour du stockage et du CRUD souple. Cassandra, des colonnes larges, pour de l'écriture massive et des time-series. Neo4j, du graphe, pour des relations. Redis, du clé-valeur en RAM, pour du cache et de l'accès par clé. OpenSearch, un moteur de recherche et d'analytics sur index inversé, donc du full-text plus des agrégations. Le critère unique : la forme de la donnée croisée avec le type de requête dominant.

**Q8. La démo est sur OpenSearch mais les slides parlent d'Elasticsearch : ça change la comparaison NoSQL ?**

Non, rien du tout. OpenSearch est un fork d'ES (par AWS, en 2021), avec la même base Lucene, le même Query DSL, les mêmes agrégations. Donc même nature, même raisonnement de choix. La différence est sur la licence et la gouvernance, et ça se joue contre Elasticsearch, pas contre les autres NoSQL.

---

## C. Architecture Big Data

**Q1. Pourquoi pas tout dans Hadoop/Hive ?**

Parce que ce n'est pas le même besoin. Hadoop et Hive, c'est du batch : on requête sur des To d'historique, la réponse arrive en minutes ou en heures, sur un modèle full-scan. ES, c'est du near-real-time : index inversé, recherche et agrégations en millisecondes ou en secondes. Pendant un incident, l'ops qui cherche les 404 des 5 dernières minutes ne peut pas attendre un job MapReduce. Donc on combine, on ne remplace pas.

**Q2. Sqoop ingère vos 14 000 logs dans OpenSearch ? (le piège)**

NON. Sqoop, c'est du transfert batch entre une base relationnelle (JDBC) et Hadoop/HDFS, et il lui faut un schéma SQL. Or des logs, c'est un flux continu semi-structuré, ça ne vient pas d'une base SQL. Pour ingérer des logs, on prend Filebeat ou Fluentd, Logstash pour le parsing, Kafka comme tampon. Sur la slide, Sqoop illustre l'ingestion depuis des bases relationnelles, pas le pipeline de logs.

**Q3. Le pipeline complet d'un log, du serveur jusqu'au dashboard ?**

Le serveur web écrit ses logs, Filebeat les lit, puis Kafka sert de tampon et de découplage. Sur le chemin chaud, Logstash ou Spark Streaming parse et enrichit (géoloc de l'IP, user-agent), indexe dans OpenSearch, et ça finit au dashboard. Sur le chemin froid, les logs bruts vont dans HDFS pour l'archivage, et Spark ou Hive gèrent l'historique. La brique de trop ici, c'est Sqoop, puisqu'il n'y a pas de base relationnelle en source.

**Q4. Batch, streaming, near-real-time : les définitions et où ça se place ?**

Le **batch**, c'est un gros lot traité en différé, ça optimise le débit : Hadoop/MapReduce, Hive, Spark batch. Le **streaming**, c'est au fil de l'eau : Kafka pour le transport, Spark Structured Streaming ou Flink. Le **near-real-time**, c'est disponible très vite après l'arrivée, mais pas instantané : c'est ES/OpenSearch, avec un refresh d'environ 1s. À retenir, ES n'est PAS du temps réel strict. Et autre nuance : Spark Streaming fait du micro-batch, alors que Flink traite événement par événement.

**Q5. La stack ELK et ses équivalents OpenSearch ?**

ELK, c'est Elasticsearch (stockage et recherche), Logstash (ingestion et transfo), Kibana (visu), plus Beats en amont. Les équivalents OpenSearch : Elasticsearch devient OpenSearch, Kibana devient OpenSearch Dashboards, et Logstash devient Data Prepper (Logstash marche aussi via un output plugin). Attention, "OpenSearch Beats" n'existe pas sous ce nom : la collecte se fait via Fluent Bit, Fluentd ou OpenTelemetry.

**Q6. ELK, c'est une architecture Lambda ?**

Non. ELK est un pipeline mono-chemin : ingestion, indexation, visu. L'architecture **Lambda**, elle, a un batch layer (Hadoop/Spark, des vues exactes mais avec de la latence), un speed layer (Kafka plus Spark/Flink, en temps réel), et un serving layer, et c'est justement là qu'ES s'insère. Beaucoup vont d'ailleurs vers **Kappa** (tout en streaming via Kafka) pour éviter d'avoir à maintenir deux pipelines.

**Q7. Pourquoi garder Hadoop ET ES ? Pourquoi pas tout dans ES ?**

C'est l'arbitrage vitesse contre coût, donc chaud/froid. ES est rapide mais cher (RAM, SSD, index inversé, réplicas). HDFS, c'est du disque banal, bas coût au To, mais lent. Donc le chaud (jours, semaines) va dans OpenSearch, et le froid (l'historique) dans HDFS. Chacun là où il est le meilleur.

**Q8. ES, c'est une base de données ? Une base principale ?**

Ça ressemble à une base documentaire (JSON, scale, agrégations), mais ce n'est pas un système de vérité. Pas d'ACID multi-docs, c'est optimisé pour la lecture et la recherche plutôt que pour des writes transactionnels, et le mapping est pénible à faire évoluer. Donc la source de vérité reste dans une vraie base, et ES n'est qu'un index secondaire de recherche alimenté par elle. On peut reconstruire l'index depuis la source, pas l'inverse.

---

## D. Licences et coûts

**Q1. Elasticsearch est-il vraiment open source ?**

Ça dépend de l'année. De 2010 à 2021, c'était de l'Apache 2.0, du vrai open source OSI. En 2021 (v7.11), abandon d'Apache 2.0 pour une double licence SSPL plus Elastic License v2, ni l'une ni l'autre validée OSI, donc du "source-available". En 2024 (v8.16), ajout de l'AGPL v3, approuvée OSI, donc de nouveau open source via cette option. Bref : vrai entre 2010 et 21, faux entre 21 et 24, de nouveau vrai depuis 24. OpenSearch, lui, est resté en Apache 2.0 sans interruption.

**Q2. La différence ES / OpenSearch, et pourquoi le fork ?**

Tout part du conflit entre Elastic et AWS, parce qu'AWS vendait du managé Elasticsearch. En janvier 2021, Elastic change de licence pour bloquer ça. AWS forke alors la dernière version sous Apache 2.0 (Elasticsearch 7.10.2) et crée OpenSearch, avec OpenSearch Dashboards en fork de Kibana, sous Apache 2.0. La base est commune (Lucene, index inversé, Query DSL), et les deux divergent depuis 2021. À noter, il y a eu aussi un litige sur la marque "Elasticsearch".

**Q3. SSPL, Elastic License v2, Apache 2.0, AGPL : quelles différences ?**

- **Apache 2.0** : permissive, OSI, quasiment aucune contrainte. C'est celle d'OpenSearch.
- **SSPL** (créée par MongoDB) : copyleft agressif, parce qu'offrir le produit en SaaS oblige à publier toute la stack de service. Du coup, refusée par l'OSI.
- **Elastic License v2** : source-available, quasi propriétaire, pas de service managé concurrent autorisé.
- **AGPL v3** : libre, OSI, avec un copyleft "réseau", mais des obligations jugées raisonnables, donc open source.

En résumé : Apache 2.0 et AGPL, c'est de l'open source. SSPL et ELv2, c'est du source-available.

**Q4. Si Elastic a remis l'AGPL en 2024, le fork ne sert plus à rien ?**

Non. D'abord une question de confiance : Elastic a changé unilatéralement une fois, il peut recommencer. Ensuite la gouvernance : OpenSearch est sous la Linux Foundation depuis 2024, alors qu'Elasticsearch reste piloté par une seule entreprise. Et puis l'écosystème : les deux divergent depuis 2021, OpenSearch a sa propre base d'utilisateurs. D'ailleurs, le retour de l'AGPL est justement une réaction stratégique, parce qu'OpenSearch leur prenait des parts de marché.

**Q5. Apache 2.0 plus gratuit, donc déployer ne coûte rien ?**

Le logiciel est gratuit, mais l'exploitation, non. Le coût, c'est l'infra (RAM, stockage), les réplicas qui doublent le disque, le CPU pour les agrégations et le scoring, et surtout l'humain : le dimensionnement, le monitoring, les montées de version. C'est justement ce modèle qui explique le conflit de licences : le moteur est gratuit, et les revenus viennent du managé et du support.

**Q6. Combien de RAM ? Y a-t-il une limite ?**

Deux règles. Première : allouer environ 50 % de la RAM au heap JVM, et laisser l'autre moitié à l'OS, parce que Lucene s'appuie sur le filesystem cache. Deuxième : il y a un plafond autour de 31-32 Go de heap. Sous ce seuil, la JVM utilise les compressed oops, des pointeurs 32 bits. Au-delà, elle passe en pointeurs 64 bits, et 40 Go peut alors donner moins de mémoire utile que 31. Conclusion : mieux vaut plusieurs nœuds à environ 31 Go qu'un seul à 64.

**Q7. Où partent les coûts à l'échelle ?**

Côté **stockage** : l'index inversé, plus les doc_values (en colonne, pour les tris et agrégations), plus le `_source`, ça fait déjà plus que les données brutes. Et les réplicas multiplient ça par au moins 2. Côté **calcul** : les agrégations et le tri (CPU et mémoire, surtout en forte cardinalité). La parade, c'est le hot-warm-cold-frozen : le récent sur des SSD chers, l'ancien vers du stockage moins cher ou du S3, et on baisse les réplicas sur les vieux index.

**Q8. Le risque le plus coûteux en prod ?**

Le mauvais dimensionnement des shards. Trop de shards (l'over-sharding), et c'est du surcoût en heap et en métadonnées. Trop peu, et on perd le parallélisme. Et comme on ne peut pas changer le nombre de primaires sans réindexer, le problème reste. Sous-dimensionné, ça donne du GC et des nœuds qui tombent. Sur-dimensionné, c'est de la facture cloud gaspillée. Le vrai coût, c'est une mauvaise décision d'archi qu'on traîne longtemps.

---

## E. Démo et optimisation

**Q1. En interne, term contre match : que se passe-t-il ? Et le score ?**

`term response=404`, c'est une question binaire : ça correspond ou pas. En filter context (sous bool/filter), il n'y a aucun score, et le résultat est cacheable dans le node query cache. `match tags=error`, c'est du query context : ça passe par l'analyzer et ça calcule un `_score` BM25 (fréquence du terme, rareté via l'IDF, longueur du champ), avec un tri par pertinence. Une honnêteté à avoir : au top-level, mon `term` renvoie un score constant de 1.0. Pour profiter du cache, il faut l'envelopper dans un `bool/filter`.

**Q2. Pourquoi un `term` sur un champ texte renvoie 0 résultat ?**

Parce que `term` fait une comparaison exacte sans analyzer. Or un champ `text` est analysé à l'indexation (minuscules plus tokenisation), donc "Error" est stocké comme le token "error". Du coup `term: { tags: "Error" }` cherche littéralement "Error" et renvoie 0 résultat, alors qu'un `match` fonctionnerait, lui, puisqu'il ré-analyse. C'est pour ça qu'on a le sous-champ `.keyword` (non analysé), pour faire du term exact sur du texte. La règle : term sur du keyword ou du numérique, match sur du text.

**Q3. `response=404` : term ou match ? Et un term sur `message` ?**

Sur `response` (une valeur exacte, un code HTTP), c'est term : on veut un filtre oui/non, pas un score. Un `match` marcherait, mais il partirait inutilement en query context. Par contre, sur `message` (du text analysé), un term serait une erreur, il faut `match`. Le bon opérateur dépend du type et du mapping du champ, pas de l'habitude.

**Q4. multi_match "opensearch" qui renvoie 5101/14000 : pertinent ou bruit ?**

Ce n'est pas faux, mais c'est trompeur. Le dataset simule du trafic vers les artefacts OpenSearch, donc "opensearch" est partout, dans les URLs comme dans les messages. Le champ `url` est analysé, donc découpé en tokens, et "opensearch" y apparaît plusieurs fois, ce qui gonfle les matches. Mais contrairement au term et au range, le multi_match trie par `_score` BM25 : les plus pertinents remontent. Au client, je dirais : "5101 documents contiennent le terme, et les plus pertinents sont en haut". Pour réduire le bruit : un top-N, un `minimum_score`, un boost `message^2`, ou un `match_phrase`.

**Q5. Filter context et cache : où placer quoi en prod ?**

Sur des logs, les contraintes répétitives (range sur timestamp, term sur response, sur service) vont en filter context, dans la clause `filter` d'un bool : pas de score, et le résultat est caché en bitset dans le node query cache. Un dashboard qui rejoue "les dernières 24h" 200 fois réutilise le bitset, donc c'est quasiment gratuit. Le query context (le `must` avec match) ne reçoit QUE la partie où la pertinence compte. Le pattern : `bool: { must: [match message], filter: [range timestamp, term response] }`. Comme ça, on ne paie le BM25 que sur le sous-ensemble déjà réduit.

**Q6. Agrégation `terms` : quel coût ? Pourquoi `size:0` ? Le danger ?**

`terms` lit les doc_values (en colonne, sur disque ou off-heap, pas sur le heap, attention). `size: 0` veut dire "pas de docs de détail, juste l'agrégation", ce qui économise du réseau et du CPU. Le danger, c'est la cardinalité : un `terms` sur `response` (5-6 valeurs) est trivial, mais sur `ip` ou un ID unique (forte cardinalité), ça fait des millions de buckets, avec un risque de saturation et du `doc_count_error` sur les top-N distribués. La parade : borner `size`, utiliser l'agg `cardinality` (du HyperLogLog, donc approximatif), ou `composite` pour paginer.

**Q7. Comment ES répond en millisecondes sur 14 000 docs ? Et à 14 millions ?**

À 14 000, presque tout serait rapide, donc la vraie question, c'est le scaling. D'abord l'index inversé : la posting list est pré-calculée, et le coût est proportionnel au nombre de termes cherchés, pas à la taille du dataset. Ensuite le sharding : l'index est découpé en shards sur plusieurs nœuds, la requête tourne en parallèle, et le coordinateur fusionne. Puis les réplicas, qui répartissent la charge en lecture. Et par-dessus, le filter cache et les doc_values. Mais ça ne reste rapide que si le sharding est bien dimensionné : le scaling se conçoit, il n'est pas gratuit.

**Q8. Les requêtes sont-elles identiques sur ES ? Un piège côté scoring ou licence ?**

Oui. OpenSearch part d'Elasticsearch 7.10.2 (Apache 2.0), donc même Query DSL (term, range, match, multi_match, bool, agrégations) et même BM25 par défaut. Les pièges : d'une part, les deux projets divergent depuis 2021 (les versions, les plugins, le ML et la sécurité diffèrent), donc ce n'est pas 100 % compatible sur les features récentes. D'autre part, la compat annoncée est avec ES 7.10, pas avec les versions 8.x. Bref : ES, c'est le concept et l'origine, OpenSearch, c'est l'implémentation libre, et c'est le même moteur Lucene en dessous.

---

*Fin. Bonne chance pour l'oral.*
