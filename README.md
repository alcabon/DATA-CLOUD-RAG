# DATA-CLOUD-RAG

Voici une vue d'ensemble de l'architecture des requêtes Salesforce Data Cloud, puis je détaille chaque couche.Voici le détail complet de chaque couche.

<img width="1472" height="1240" alt="image" src="https://github.com/user-attachments/assets/7f616a29-4a41-4131-8182-30e524e27029" />

---

## 1. Les objets DML requêtés : DLO vs DMO

**Data Lake Objects (DLO)** sont le stockage brut. On ne les requête généralement pas directement depuis les requêtes métier — ils servent à la transformation.

**Data Model Objects (DMO)** sont les objets cibles des requêtes. Ils suivent le schéma unifié Salesforce (basé sur le modèle Customer 360) :

- `UnifiedIndividual__dlm` — l'individu réconcilié (merge des identités)
- `ContactPointEmail__dlm`, `ContactPointPhone__dlm` — points de contact
- `SalesOrder__dlm`, `Opportunity__dlm` — objets transactionnels
- Custom DMOs suffixés en `__dlm`

Une requête SOQL Data Cloud ressemble à :
```sql
SELECT UnifiedIndividual__c.Id__c, Email__c
FROM UnifiedIndividual__dlm
WHERE FirstName__c = 'Marie'
USING SCOPE segment__c = 'SegmentVIP'
```

---

## 2. L'index et le chunk (Knowledge / RAG)

Quand des données Knowledge (articles, PDFs, documents) sont chargées dans Data Cloud pour Agentforce, elles passent par un pipeline en deux étapes.

**Le chunking** découpe chaque document en fragments de texte (`chunk_text`) avec :
- `chunk_id` — identifiant unique du fragment
- `chunk_number` — position ordinale dans le document source
- `source_id` — FK vers le DMO parent (ex. `KnowledgeArticle__dlm`)
- un token overlap configurable (souvent 10-20% chevauchement entre chunks)

**L'index vectoriel** stocke l'embedding de chaque chunk et permet la recherche sémantique :
- modèle d'embedding : Ada-002 (OpenAI) ou le modèle interne Salesforce
- métrique : `cosine` ou `dot product`
- moteur : Approximate Nearest Neighbor (ANN)
- paramètre `top_k` : nombre de chunks renvoyés

Une requête vector search ressemble à :
```sql
SELECT chunk_id, chunk_text, SIMILARITY_SCORE()
FROM EinsteinSearchIndex__dlm
WITH VECTOR INDEX = 'KnowledgeIndex'
WHERE SIMILARITY(chunk_text, :queryVector) > 0.75
ORDER BY SIMILARITY_SCORE() DESC
LIMIT 5
```

---

## 3. Comment savoir comment les données Knowledge ont été chargées ?

Plusieurs endroits à inspecter :

**Via l'UI Data Cloud :**
- `Data Cloud Setup → Data Streams` : liste tous les connecteurs actifs avec leur fréquence et statut de synchronisation
- `Data Cloud Setup → Data Lake Objects` : voir le schéma brut et les métadonnées de chargement (`LastRefreshedDate`, `RecordCount`)
- `Einstein Studio → Vector Databases` : voir les index créés, le modèle d'embedding utilisé, la taille des chunks, et la date de dernière indexation

**Via requête :**
```sql
-- Inspecter les métadonnées d'un flux
SELECT Name__c, Status__c, LastRunDate__c, RecordCount__c
FROM DataStream__dlm

-- Voir les chunks d'un article Knowledge spécifique
SELECT chunk_id, chunk_number, chunk_text, source_id
FROM EinsteinChunk__dlm
WHERE source_id = 'ka0XXXXXXXXX'
ORDER BY chunk_number ASC
```

**Via Apex/API :**
```apex
ConnectApi.DataCloudQueryResult result = 
    ConnectApi.DataCloud.runQuery(orgId, 
        'SELECT * FROM DataStream__dlm WHERE Type__c = \'Knowledge\'');
```

---

## 4. Récapitulatif de la hiérarchie

| Niveau | Objet | Rôle |
|--------|-------|------|
| Ingestion | Data Stream | Connecteur source |
| Stockage | DLO (`__dlo`) | Données brutes partitionnées |
| Modèle | DMO (`__dlm`) | Schéma unifié requêtable |
| Sémantique | Chunk (`EinsteinChunk__dlm`) | Fragment de texte |
| Recherche | Vector Index | ANN sur embeddings |
| Requête | SOQL / Vector Search | Interface consommatrice |

Le point clé : dans Data Cloud, **on ne requête jamais les DLO directement en production** — on passe toujours par les DMO, et pour le RAG on joint DMO → Chunk → Index vectoriel.

---

Voici une vue d'ensemble de l'architecture des requêtes Salesforce Data Cloud, puis je détaille chaque couche.Voici le détail complet de chaque couche.

---

## 1. Les objets DML requêtés : DLO vs DMO

**Data Lake Objects (DLO)** sont le stockage brut. On ne les requête généralement pas directement depuis les requêtes métier — ils servent à la transformation.

**Data Model Objects (DMO)** sont les objets cibles des requêtes. Ils suivent le schéma unifié Salesforce (basé sur le modèle Customer 360) :

- `UnifiedIndividual__dlm` — l'individu réconcilié (merge des identités)
- `ContactPointEmail__dlm`, `ContactPointPhone__dlm` — points de contact
- `SalesOrder__dlm`, `Opportunity__dlm` — objets transactionnels
- Custom DMOs suffixés en `__dlm`

Une requête SOQL Data Cloud ressemble à :
```sql
SELECT UnifiedIndividual__c.Id__c, Email__c
FROM UnifiedIndividual__dlm
WHERE FirstName__c = 'Marie'
USING SCOPE segment__c = 'SegmentVIP'
```

---

## 2. L'index et le chunk (Knowledge / RAG)

Quand des données Knowledge (articles, PDFs, documents) sont chargées dans Data Cloud pour Agentforce, elles passent par un pipeline en deux étapes.

**Le chunking** découpe chaque document en fragments de texte (`chunk_text`) avec :
- `chunk_id` — identifiant unique du fragment
- `chunk_number` — position ordinale dans le document source
- `source_id` — FK vers le DMO parent (ex. `KnowledgeArticle__dlm`)
- un token overlap configurable (souvent 10-20% chevauchement entre chunks)

**L'index vectoriel** stocke l'embedding de chaque chunk et permet la recherche sémantique :
- modèle d'embedding : Ada-002 (OpenAI) ou le modèle interne Salesforce
- métrique : `cosine` ou `dot product`
- moteur : Approximate Nearest Neighbor (ANN)
- paramètre `top_k` : nombre de chunks renvoyés

Une requête vector search ressemble à :
```sql
SELECT chunk_id, chunk_text, SIMILARITY_SCORE()
FROM EinsteinSearchIndex__dlm
WITH VECTOR INDEX = 'KnowledgeIndex'
WHERE SIMILARITY(chunk_text, :queryVector) > 0.75
ORDER BY SIMILARITY_SCORE() DESC
LIMIT 5
```

---

## 3. Comment savoir comment les données Knowledge ont été chargées ?

Plusieurs endroits à inspecter :

**Via l'UI Data Cloud :**
- `Data Cloud Setup → Data Streams` : liste tous les connecteurs actifs avec leur fréquence et statut de synchronisation
- `Data Cloud Setup → Data Lake Objects` : voir le schéma brut et les métadonnées de chargement (`LastRefreshedDate`, `RecordCount`)
- `Einstein Studio → Vector Databases` : voir les index créés, le modèle d'embedding utilisé, la taille des chunks, et la date de dernière indexation

**Via requête :**
```sql
-- Inspecter les métadonnées d'un flux
SELECT Name__c, Status__c, LastRunDate__c, RecordCount__c
FROM DataStream__dlm

-- Voir les chunks d'un article Knowledge spécifique
SELECT chunk_id, chunk_number, chunk_text, source_id
FROM EinsteinChunk__dlm
WHERE source_id = 'ka0XXXXXXXXX'
ORDER BY chunk_number ASC
```

**Via Apex/API :**
```apex
ConnectApi.DataCloudQueryResult result = 
    ConnectApi.DataCloud.runQuery(orgId, 
        'SELECT * FROM DataStream__dlm WHERE Type__c = \'Knowledge\'');
```

---

## 4. Récapitulatif de la hiérarchie

| Niveau | Objet | Rôle |
|--------|-------|------|
| Ingestion | Data Stream | Connecteur source |
| Stockage | DLO (`__dlo`) | Données brutes partitionnées |
| Modèle | DMO (`__dlm`) | Schéma unifié requêtable |
| Sémantique | Chunk (`EinsteinChunk__dlm`) | Fragment de texte |
| Recherche | Vector Index | ANN sur embeddings |
| Requête | SOQL / Vector Search | Interface consommatrice |

Le point clé : dans Data Cloud, **on ne requête jamais les DLO directement en production** — on passe toujours par les DMO, et pour le RAG on joint DMO → Chunk → Index vectoriel.

---

Exactement, et c'est le cœur du problème. Laissez-moi illustrer pourquoi les techniques standard échouent sur la connaissance non structurée.## Pourquoi le SOQL standard est structurellement inadapté à Knowledge

<img width="1472" height="1280" alt="image" src="https://github.com/user-attachments/assets/2eac5384-8a91-41d1-82c1-b907f0584445" />


### Le problème du champ `Body`

Dans Salesforce Knowledge standard, le contenu réel d'un article vit dans un champ `Body` ou dans des champs rich-text (`__c` de type Long Text Area). Ces champs posent trois problèmes immédiats :

**Ils ne sont pas indexés pour la recherche full-text en SOQL.** Un `WHERE Body LIKE '%remboursement%'` fait un scan séquentiel qui explose en performance dès que la base dépasse quelques milliers d'articles, et qui ne passe pas du tout en Data Cloud.

**Ils contiennent du HTML brut.** Le moteur SOQL ne parse pas le HTML — `LIKE '%remboursement%'` peut rater une occurrence encapsulée dans `<p class="section"><strong>Remboursement</strong>`.

**La limite de 255 caractères** sur les champs indexables en SOQL standard exclut de facto tout le contenu long-form.

### Le problème sémantique

Même avec SOSL (Salesforce Object Search Language, qui fait du full-text réel), on reste bloqué sur la **correspondance lexicale**. Si un agent tape "comment déclarer un sinistre", SOSL cherche littéralement ces mots. Il ne retrouvera pas un article qui parle de "procédure de déclaration de dommages" ou "démarche suite à un accident".

C'est là le fossé fondamental : **la requête exprime une intention, le document exprime une réponse dans un vocabulaire différent**.

### Ce que le RAG résout concrètement dans Data Cloud

Le pipeline pré-ingestion fait trois choses que SOQL ne fera jamais :

**Strip et normalisation** — le HTML est nettoyé, les balises retirées, le texte structuré extrait proprement avant même d'entrer dans l'index.

**Chunking intelligent** — un article de 20 pages n'est pas stocké comme un bloc. Il est découpé en fragments de ~512 tokens avec un overlap (chevauchement) de ~10% entre chunks adjacents, de sorte qu'une idée qui chevauche deux paragraphes ne soit jamais perdue à la frontière d'un découpage.

**Espace vectoriel partagé** — la requête utilisateur ET les chunks sont projetés dans le même espace vectoriel (1536 dimensions pour Ada-002). La distance cosinus entre le vecteur-requête et le vecteur-chunk mesure la **proximité sémantique**, pas la correspondance lexicale. "Sinistre" et "dommage" se retrouvent proches dans cet espace parce qu'ils apparaissent dans des contextes similaires dans les corpus d'entraînement.

### La subtilité du score de similarité

Ce que le RAG apporte en plus c'est un **score ordonnable**. Là où SOQL retourne une liste plate sans discrimination, la vector search retourne des chunks avec un score `SIMILARITY_SCORE()` entre 0 et 1. On peut donc :

- filtrer sous un seuil (ex. rejeter tout chunk < 0.72) pour éviter le bruit
- pondérer plusieurs sources (`chunk` récent vs `chunk` ancien)
- fusionner des chunks de plusieurs articles différents pour construire une réponse synthétique — quelque chose qu'une requête SOQL ne peut pas faire du tout

C'est exactement pour ça qu'Agentforce utilise ce pipeline RAG plutôt qu'une simple recherche Knowledge standard : la pertinence de la réponse de l'agent dépend directement de la qualité des chunks récupérés, pas du titre de l'article.

---

Exactly — and it's a hard platform constraint, not a configuration issue.It's a hard engine-level constraint. The Salesforce query optimizer simply **refuses to build a predicate** on any field backed by a Long Text Area column — you get a compile-time error, not a runtime performance issue:

```
System.QueryException: field 'Body' can not be filtered in a WHERE clause
```

This is by design: Long Text Areas are stored off-row (as BLOBs in a separate storage layer), so there is no B-tree index for the query planner to walk. The same applies to rich-text fields in Knowledge (`VersionData`, `Body`, any `__c` of type Long Text Area).

**SOSL** (`FIND`) is the partial escape hatch — it does full-text tokenization and has its own inverted index — but it still has real limits:

- lexical matching only, no semantic similarity
- no scoring you can `ORDER BY` in a deterministic way
- `RETURNING` clause limits which fields come back
- cross-object result sets are awkward to join back
- still misses synonyms, paraphrases, and multilingual variants

So the gap that RAG fills is actually two separate gaps stacked on top of each other: first the **structural gap** (can't filter the field at all in SOQL), then the **semantic gap** (even if you could, lexical matching wouldn't find the right content). That's why the chunking + vector index approach isn't just a nice-to-have — it's the only viable path for content retrieval on Knowledge article bodies at scale.
