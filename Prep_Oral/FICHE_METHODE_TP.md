# Fiche méthode TP : requêtes OpenSearch / Elasticsearch

Fiche de révision basée sur les erreurs réelles faites pendant les TP, avec la correction et un exemple minimal pour chaque cas.

Les données de référence :
- Index `movies` : champs sous `fields.` (ex. `fields.title`, `fields.directors`, `fields.genres`, `fields.rating`, `fields.year`, `fields.release_date`, `fields.plot`).
- Index `logs` : champs `response`, `timestamp`, `tags`, `message`, `url`, `ip`.

---

## 1. Le squelette d'une requête

Une requête de recherche, c'est toujours la même structure : un verbe HTTP, un index, l'endpoint `_search`, puis un corps en JSON.

```
GET movies/_search
{
  "query": {
    "match": { "fields.title": "Star Wars" }
  }
}
```

Trois choses à retenir :
- `GET ...` sur la première ligne, puis le JSON juste en dessous.
- Tout est du JSON : des accolades, des guillemets droits, des virgules entre les éléments.
- On parle à OpenSearch via une API REST, donc c'est le même principe que du HTTP.

---

## 2. Les pièges de syntaxe (erreurs vécues)

### Erreur : "EXPECTED ONE OF GET/POST/PUT..."

Cause : il y a du texte ou du JSON AVANT le `GET` dans la cellule. Le moteur attend un verbe HTTP en premier, il tombe sur autre chose.

Mauvais (du texte traîne avant) :
```
voici ma requête
GET movies/_search
{ "query": { "match_all": {} } }
```

Correct (la cellule commence par le verbe) :
```
GET movies/_search
{ "query": { "match_all": {} } }
```

Règle : une cellule = une seule requête, qui commence par `GET` (ou `POST`, `PUT`, `DELETE`). Pas de commentaire ni de texte avant.

### Erreur : mettre une chaîne au lieu d'un objet dans `must`

Cause : `must` (et `should`, `must_not`, `filter`) attend une LISTE d'OBJETS requête, pas le mot-clé tout seul.

Mauvais :
```
{ "query": { "bool": { "must": [ "terms" ] } } }
```

Correct (chaque élément est un objet `{ type_de_requête: {...} }`) :
```
{
  "query": {
    "bool": {
      "must": [
        { "term": { "fields.year": 2013 } }
      ]
    }
  }
}
```

### Erreur : présentation en cellule "raw" sans coloration

Cause : une cellule de type `raw` n'a pas de coloration syntaxique, c'est illisible.

Correction : écrire les requêtes dans une cellule markdown, dans un bloc clôturé par trois backticks suivis de `json`. La requête s'affiche en couleurs.

---

## 3. Les types de requêtes (un exemple chacun)

### term : valeur EXACTE, pas d'analyse

Pour les nombres, les booléens, les champs `keyword`.

```
GET movies/_search
{ "query": { "term": { "fields.year": 2013 } } }
```

### match : recherche plein texte (analysée)

Pour chercher un mot dans du texte. La recherche est mise en minuscule et découpée comme le texte indexé.

```
GET movies/_search
{ "query": { "match": { "fields.title": "star wars" } } }
```

### range : intervalle (nombres ou dates)

Opérateurs : `gte` (>=), `gt` (>), `lte` (<=), `lt` (<).

```
GET movies/_search
{ "query": { "range": { "fields.rating": { "gte": 8 } } } }
```

Deux bornes sur le même champ se mettent dans le MÊME objet range :
```
{ "query": { "range": { "fields.rating": { "gt": 8, "lte": 10 } } } }
```

### bool : combiner plusieurs conditions

- `must` : ET (toutes les conditions, comptent dans le score).
- `must_not` : NON (exclut).
- `should` : OU (au moins une, augmente le score).
- `filter` : ET mais sans score, et cacheable (idéal pour les filtres exacts).

```
GET movies/_search
{
  "query": {
    "bool": {
      "must":     [ { "match": { "fields.title": "Star" } } ],
      "must_not": [ { "match": { "fields.genres": "Adventure" } } ],
      "filter":   [ { "range": { "fields.rating": { "gte": 8 } } } ]
    }
  }
}
```

### multi_match : un mot dans PLUSIEURS champs

Utile quand on ne sait pas dans quel champ se trouve l'information.

```
GET movies/_search
{
  "query": {
    "multi_match": {
      "query": "Jersey",
      "fields": [ "fields.title", "fields.plot" ]
    }
  }
}
```

### exists : le champ est présent

```
GET movies/_search
{ "query": { "exists": { "field": "fields.rating" } } }
```

---

## 4. term contre match contre .keyword (le point de confusion)

Deux moments différents transforment les données :
- `term` / `match` agissent sur la RECHERCHE.
- `text` / `.keyword` agissent sur la donnée STOCKÉE.

Au stockage, un champ `text` est analysé : "George Lucas" devient les tokens `[george, lucas]` en minuscule. La valeur entière n'est plus stockée telle quelle.

`term` ne transforme pas la recherche, donc il cherche la valeur exacte. Sur un champ `text`, il ne trouve pas la valeur entière (elle a été découpée).

Erreur classique :
```
{ "query": { "term": { "fields.directors": "George Lucas" } } }   // 0 résultat
```

Correction, viser le sous-champ `.keyword` (la valeur brute, non analysée) :
```
{ "query": { "term": { "fields.directors.keyword": "George Lucas" } } }   // trouve
```

Résumé : `term` sur un champ `keyword` (ou numérique), `match` sur un champ `text`. Le sous-champ `.keyword` existe justement pour faire du `term` exact sur du texte.

---

## 5. Le piège du bon champ (erreurs directors / tags)

### Erreur : mauvais nom de champ

Pendant le TP, recherche sur `fields.actors.keyword` alors qu'il fallait `fields.directors.keyword`. Résultat faux ou vide.

Réflexe : avant d'écrire une requête, regarder un document pour connaître les vrais noms de champs.
```
GET movies/_search
{ "size": 1 }
```

### Erreur : chercher un mot dans un champ qui ne le contient pas

Pendant la démo logs, `match` sur `message` pour "error" a renvoyé 0 résultat. Le champ `message` contient du log Apache (`GET /... HTTP/1.1 200`), le mot "error" n'y est pas. Il était dans le champ `tags`.

Mauvais (0 résultat) :
```
{ "query": { "match": { "message": "error" } } }
```

Correct (le mot est dans tags) :
```
{ "query": { "match": { "tags": "error" } } }
```

Règle : quand une requête renvoie 0 résultat alors que tu es sûr que la donnée existe, le problème est presque toujours (1) le mauvais champ, ou (2) `term` sur un champ analysé.

---

## 6. Les agrégations (un exemple chacun)

Pour une agrégation pure, on met `"size": 0` pour ne pas ramener les documents, juste le calcul.

### histogram : regrouper des nombres par tranche

Erreur vécue : oublier `interval`, ce qui donne une erreur. Une `histogram` a TOUJOURS besoin d'un `interval`.

```
GET movies/_search
{
  "size": 0,
  "aggs": {
    "par_note": {
      "histogram": { "field": "fields.rating", "interval": 1 }
    }
  }
}
```

### date_histogram : regrouper par période de temps

```
GET movies/_search
{
  "size": 0,
  "aggs": {
    "par_mois": {
      "date_histogram": { "field": "fields.release_date", "calendar_interval": "month" }
    }
  }
}
```

### terms : regrouper par valeur (le "group by")

On agrège sur un champ exact, donc le sous-champ `.keyword`.

```
GET movies/_search
{
  "size": 0,
  "aggs": {
    "par_genre": {
      "terms": { "field": "fields.genres.keyword" }
    }
  }
}
```

### sous-agrégation : une agrégation dans une autre

Exemple : la distribution par note, et dans chaque tranche, la répartition par genre. La clé est le `aggs` imbriqué.

```
GET movies/_search
{
  "size": 0,
  "aggs": {
    "par_note": {
      "histogram": { "field": "fields.rating", "interval": 1 },
      "aggs": {
        "par_genre": {
          "terms": { "field": "fields.genres.keyword" }
        }
      }
    }
  }
}
```

---

## 7. Checklist de debug (quand ça ne marche pas)

1. La cellule commence-t-elle bien par `GET` (rien avant) ?
2. Le JSON est-il valide (accolades fermées, virgules entre les éléments, guillemets droits) ?
3. Dans un `bool`, chaque élément de `must` / `should` / `filter` est-il un objet `{ type: {...} }` et pas une chaîne ?
4. 0 résultat ? Vérifier le nom exact du champ (`GET index/_search { "size": 1 }`) et vérifier que le mot est bien dans ce champ.
5. Recherche exacte qui échoue sur du texte ? Utiliser le sous-champ `.keyword`, ou passer à `match`.
6. Agrégation `histogram` ? Ne pas oublier `interval`.
7. Agrégation sans documents voulus ? Ajouter `"size": 0`.

---

## 8. Antisèche term / match / agrégation

| Je veux... | J'utilise | Sur quel champ |
|------------|-----------|----------------|
| Une valeur exacte (nombre, code) | `term` | numérique ou `.keyword` |
| Un mot dans du texte | `match` | champ `text` |
| Un mot dans plusieurs champs | `multi_match` | plusieurs champs `text` |
| Un intervalle | `range` | numérique ou date |
| Combiner des conditions | `bool` (must / must_not / should / filter) | selon chaque sous-requête |
| Compter par valeur | `terms` (agg) | `.keyword` |
| Distribution par tranche de nombre | `histogram` (agg) + `interval` | numérique |
| Distribution dans le temps | `date_histogram` (agg) | date |
