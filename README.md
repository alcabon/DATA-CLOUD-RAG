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
