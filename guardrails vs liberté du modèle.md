C'est une question très pertinente qui touche à un vrai débat dans l'écosystème des outils AI pour le développement.

## La tension réelle : guardrails vs liberté du modèle

Tu soulèves quelque chose d'important : **les LLMs de dernière génération ont effectivement absorbé énormément de connaissance contextuelle** (conventions Salesforce, patterns LWC, limites governor, best practices Apex...). Un Sonnet 4.6 ou Opus 4.8 dans Claude Code va souvent produire du code correct *sans avoir besoin* qu'on lui rappelle "n'oublie pas les bulkification patterns" ou "attention aux SOQL dans les loops".

---

## Quand les agent.md / skills deviennent contre-productifs

Les cas où ils **brident effectivement** le modèle :

- **Sur-spécification verbeuse** : 500 lignes d'instructions qui re-expliquent ce que le modèle sait déjà → bruit dans le contexte, confusion potentielle
- **Contraintes périmées** : une rule écrite pour GPT-3.5 qui force un comportement que Sonnet 4.6 gère mieux naturellement
- **Faux garde-fous** : "toujours ajouter un try/catch" dans un SKILL.md alors que le modèle contextualise déjà quand c'est nécessaire
- **Fragmentation du contexte** : trop de fichiers `.md` chargés = fenêtre de contexte saturée avec du méta-information plutôt que du vrai code

---

## Ce qui garde de la valeur malgré tout

Mais certains usages restent **légitimes et non-redondants** :

**Connaissance organisationnelle unique**
- Naming conventions *spécifiques à ton org* (`Acme__c`, préfixes custom, namespaces de packages)
- Architecture décisionnelle ("on n'utilise pas Platform Events pour X raison historique")
- Règles métier non-documentées ailleurs

**Contexte de codebase**
- Structure de ton repo Salesforce (`force-app/main/default/...` customisé)
- Patterns maison qui dévient du standard
- Dépendances inter-packages

**Contraintes d'environnement réelles**
- Version API cible, sandboxes disponibles, restrictions de déploiement
- Ce que le modèle ne peut *pas* inférer du code seul

**Reproductibilité d'équipe**
- Quand plusieurs devs (ou plusieurs sessions) doivent produire du code cohérent sans re-contextualiser à chaque fois

---

## La tendance de fond

Le mouvement va clairement vers **moins d'instructions prescriptives, plus de contexte factuel** :

```
❌ "Toujours écrire des classes Apex avec une séparation concerns claire"
   → Le modèle sait déjà

✅ "Notre org utilise le pattern Repository via fflib-apex-common,
    tous les DML passent par UnitOfWork"
   → Information factuelle irremplaçable
```

---

## Conclusion

Les skills/agent.md **évoluent** plutôt qu'ils ne disparaissent : leur rôle passe de **"enseigner les bonnes pratiques"** (inutile avec Sonnet 4.6+) à **"transmettre la connaissance organisationnelle non-inferable"**. Ce qui est bavard, prescriptif et générique mérite effectivement d'être supprimé — ça consomme du contexte pour zéro valeur ajoutée.

La bonne heuristique : **si l'information existe dans la documentation publique Salesforce, retire-la du SKILL.md. Si elle n'existe que dans ta tête ou ton repo, garde-la.**
