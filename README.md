# TP OpenSearch - Partie 1 : Mise en place d'OpenSearch Dashboards (Windows)

##  Objectif

- Installer et lancer **OpenSearch Dashboards** sur Windows pour visualiser et interagir avec les données de manière intuitive.

- Ajouter des données dans un index et faire des requetes CRUD

- Faire des requêtes de recherches sur Exacte sur le indexes 

- Faire des requêtes de recherches en full texte sur le indexes 


---


## I. Mise en place d'OpenSearch
### 1. Téléchargement de l’outil

1. Télécharcher le répértoire [opensearch3.0.0-windows-x64.zip](https://bul.univ-lyon2.fr/index.php/s/xKcKJpuPGWr3XWY) ou depuis le répertoire Deep BigData du réseau de l'IUT (**recommandé**) :
   👉 (votre dossier de promo) -> Deep BigData 

2. Copier le réptoire zip dans votre répertoire Documents 
   **`opensearch3.0.0-windows-x64.zip`**


---

## 2. Extraction de l’archive

1. Faites un clic droit sur l’archive téléchargée.
2. Sélectionnez **"Extraire tout..."**.
3. Essayer d'extraire dans le même répertoire.


---

### 3. Lancer OpenSearch 
#### Structure des dossiers après extraction

```
opensearch-3.0.0
├── opensearch-3.0.0
├── opensearch_dashboards-3.0.0
```

#### Méthodes d'exécution du script batch  

Il existe deux façons d'exécuter le script batch pour démarrer OpenSearch (Si une invite de commande s'ouvre et se referme rapidement, il faut passer donc à la seconde méthode) :  

##### 1. **Exécution via l'interface graphique de Windows**  
1. Accédez au répertoire principal de votre installation OpenSearch (dossier `opensearch-1.3.20`).  
2. Double-cliquez sur le fichier `opensearch-windows-install.bat`.  
   - Une invite de commande s'ouvrira avec une instance OpenSearch en cours d'exécution.  

##### 2. **Exécution depuis l'Invite de commande ou PowerShell**  
1. Ouvrez **l'Invite de commande** (tapez `cmd`) ou **PowerShell** (tapez `powershell`) dans la barre de recherche Windows.  
2. Accédez au répertoire d'installation d'OpenSearch :  
   ```cmd
   cd \chemin\vers\opensearch-3.0
   ```
3. Exécutez le script batch :  
   ```cmd
   .\opensearch-windows-install.bat
   ```

====

Dans le cas ou vous rencontrez des erreurs, voici ce qu'il faut faire 

====

1. Supprimer tout les espaces dans les noms de répertoire.
1. Dans le fichier ``opensearch.yml`` qui est dans le repertoire ``\opensearch-3.0.0\config``, rajouter ce code : 

```yaml
plugins.security.disabled: true
plugins.security.ssl.http.enabled: false
plugins.security.ssl.transport.enabled: false
```
3. Executer ces commandes dans le powershell : 
```bash
$env:JAVA_HOME = "C:\Users\<nom-de-votre-session>\Documents\OpenSearch3.0-winddows-x64\OpenSearch-3.0\opensearch-3.0.0\jdk"
```
4. Ensuite : 
```bash
$env:PATH = "$env:JAVA_HOME\bin;$env:PATH"
```

5. Exécutez le script batch depuis le dossier ``opensearch-3.0.0\bin``:  

   ```cmd
   .\opensearch-windows-install.bat
   ```

##### Vérification du bon fonctionnement  
Ouvrez une nouvelle invite de commande et envoyez des requêtes pour vérifier qu'OpenSearch est actif.  

- **Requête vers le port 9200** (authentification avec `admin:admin`) :  
  ```cmd
  curl.exe -X GET http://localhost:9200 -u "admin:admin" --insecure
  ```
  > **Remarque** : L'option `--insecure` est nécessaire car les certificats TLS sont auto-signés.  

  **Réponse attendue** :  
  ```json
  {
     "name" : "nom-de-l-hote",
     "cluster_name" : "opensearch",
     "cluster_uuid" : "7Nqtr0LrQTOveFcBb7Kufw",
     "version" : {
        "distribution" : "opensearch",
        "number" : "<version>",
        "build_type" : "<type-de-build>",
        "build_hash" : "<hash-de-build>",
        "build_date" : "<date-de-build>",
        "build_snapshot" : false,
        "lucene_version" : "<version-lucene>",
        "minimum_wire_compatibility_version" : "7.10.0",
        "minimum_index_compatibility_version" : "7.0.0"
     },
     "tagline" : "The OpenSearch Project: https://opensearch.org/"
  }
  ```

- **Liste des plugins installés** :  
  ```cmd
  curl.exe -X GET http://localhost:9200/_cat/plugins?v -u "admin:admin" --insecure
  ```
  **Exemple de réponse** :  
  ```
  nom-hote opensearch-alerting                  1.3.20
  nom-hote opensearch-anomaly-detection         1.3.20
  nom-hote opensearch-asynchronous-search       1.3.20
  [...] (autres plugins)
  ```

## II. Interagir avec OpenSearch via Python (`opensearch-py`)

### **Objectifs**  
1. Utiliser la bibliothèque `opensearch-py` pour se connecter au cluster.  
2. Exécuter des requêtes de base pour inspecter l'état du cluster.  
3. Créer un index avec des réplicas.  

### **Prérequis**  
- Un cluster OpenSearch local fonctionnel (cf. Partie 1 du TP).  
- Python 3.8+ installé.  
- VS Code (avec l'extension Jupyter) **ou** Spyder.  
- Bibliothèques à installer :  
  ```
  pip install opensearch-py pandas
  ```

---

### **Étapes à suivre**  

#### **1. Création du notebook Jupyter**  
- Ouvrez VS Code / Spyder et créez un nouveau notebook Jupyter (`.ipynb`).  
- Importez les bibliothèques nécessaires :  
  ```
  from opensearchpy import OpenSearch

  import pandas as pd
  ```

#### **2. Connexion au cluster**  
Configurez le client OpenSearch (adaptez `hosts` et `auth` si nécessaire) :  
```python
client = OpenSearch(
    hosts=[{"host": "localhost", "port": 9200}],
    http_auth=("admin", "admin"),
    use_ssl=False,
    verify_certs=False,
    ssl_show_warn=False
)
```
**Test de connexion :**
```python
print(client.info())
```

#### **3. Requêtes d'inspection du cluster**  
Répondez aux questions suivantes en exécutant des requêtes :  

##### **Question 1 : Health du cluster**  
```python
# completez moi
```

##### **Question 2 : Nombre de nœuds**  
```python
# completez moi
```

##### **Question 3 : Adresses IP des nœuds**  
```python
# completez moi
```

#### **4. Création d'un index avec réplicas**  
Créez un index nommé `mon_index` avec 2 réplicas :  
```python
# completez moi
```

#### **5. Vérification de l'index**  
```python
# completez moi
```

---

### **Lancement d'OpenSearch Dashboards**

Il existe deux méthodes pour exécuter OpenSearch Dashboards :

#### **1. Méthode graphique (via l'interface Windows)**
1. Accédez au répertoire d'installation d'OpenSearch Dashboards (dossier `opensearch-dashboards-1.3.20`).
2. *(Optionnel)* Modifiez le fichier `opensearch_dashboards.yml` dans le dossier `config` pour ajuster les paramètres.
3. Ouvrez le dossier `bin` et double-cliquez sur le fichier `opensearch-dashboards.bat`.
   - Une fenêtre de commande s'ouvrira avec l'instance Dashboards en cours d'exécution.

#### **2. Méthode en ligne de commande (CMD/PowerShell)**
1. Lancez **l'Invite de commande** (`cmd`) ou **PowerShell** via la barre de recherche Windows.
2. Placez-vous dans le répertoire d'installation :
   ```cmd
   cd \chemin\vers\opensearch-dashboards-1.3.20
   ```
3. *(Optionnel)* Modifiez les paramètres dans `config\opensearch_dashboards.yml`.
4. Exécutez le script :
   ```cmd
   .\bin\opensearch-dashboards.bat
   ```

---


**La suite de ce TP est organisé comme suit :**   


---
## III. Requetes CRUD
- Voir [TP01_CRUD.ipynb](TP01_CRUD.ipynb) 
## IV. Requetes de recherche sur texte exact
- Voir [TP02_Structured-Search.ipynb](TP02_Structured-Search.ipynb) 
## V. Requetes sur full texte
- Voir [TP03_FULLTEXT_SEARCH.ipynb](TP03_FULLTEXT_SEARCH.ipynb) 

---

## **Rendu attendu**  
- Un notebook Jupyter exécutable avec la partie 1 de la section I. :  
  - Les réponses aux 4 questions.  
- L'ensemble des parties ; CRUD, StructuredSearch et FullTextSearch

---



Modif dans la config de dashaboard

```sh


opensearch.hosts: [http://localhost:9200]
opensearch.ssl.verificationMode: none
opensearch.username: admin
opensearch.password: admin
opensearch.requestHeadersWhitelist: [authorization, securitytenant]

opensearch_security.enabled: false

```