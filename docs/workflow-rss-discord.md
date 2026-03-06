# Workflow RSS -> Discord

**Fichier** : `workflows/veille-rss-discord-medium-limit.json`

## Objectif

Agreger des articles RSS de 4 sources data engineering et envoyer un digest quotidien sur Discord. Limite Medium a 3 articles max pour eviter le spam.

## Schema du workflow

```
Schedule (8h)
    |-- RSS Octo
    |-- RSS TDS (Towards Data Science)
    |-- RSS Medium (tag data-engineering)
    |-- RSS Substack (Practical Data Engineering)
           |
           v
        Merge (4 inputs)
           |
           v
      Filter <24h
           |
           v
     Dedup & Sort
           |
           v
     Build Digest
           |
           v
     Send Discord
```

## Detail des noeuds

### Schedule

Declenchement tous les jours a 8h du matin.

### RSS Octo / TDS / Medium / Substack

4 noeuds RSS en parallele. Chacun lit un flux different :

| Noeud | URL | Theme |
|-------|-----|-------|
| RSS Octo | `blog.octo.com/feed/` | Architecture, craft logiciel |
| RSS TDS | `towardsdatascience.com/feed` | Data science, ML |
| RSS Medium | `medium.com/feed/tag/data-engineering` | Data engineering |
| RSS Substack | `practicaldataengineering.substack.com/feed` | Data engineering |

### Merge

Fusionne les 4 flux en une seule liste. Mode append, `numberInputs: 4`.

### Filter <24h

Noeud IF qui compare la date de publication a `Date.now() - 24h`. Teste dans l'ordre : `isoDate`, `pubDate`, `published`, `date`. Seuls les articles des dernieres 24h passent.

### Dedup & Sort (noeud Code JS)

C'est le coeur de la logique. Il fait 5 choses :

**1) Memoire persistante**
Utilise `$getWorkflowStaticData('global')` pour stocker les liens deja envoyes dans `store.sentLinks`. Chaque entree = `{ lien: timestamp }`.

**2) Nettoyage memoire**
Supprime les entrees de plus de 30 jours pour eviter que la memoire grossisse.

**3) Normalisation**
Pour chaque article :
- Extrait `title` (strip les tags HTML), `link`, `creator`
- Determine la source : `medium` ou `other` (selon si le lien contient `medium.com`)

**4) Deduplication + filtrage**
- Cle unique = lien (ou titre+timestamp si pas de lien)
- Si la cle existe dans `sentLinks` -> ignore

**5) Tri et limites**
- Tri par date decroissante (plus recent d'abord)
- Max 10 articles au total
- Max 3 articles Medium
- Marque les articles selectionnes comme envoyes
- Si aucun article restant -> retourne `{ noArticles: true }`

### Build Digest (noeud Code JS)

Construit le message Discord :
- Si `noArticles: true` -> "Aucun nouvel article"
- Sinon, message Markdown : numero, titre, auteur, lien pour chaque article
- Tronque a 1950 caracteres (limite Discord = 2000)

### Send Discord

POST HTTP sur `$env.DISCORD_WEBHOOK_URL` avec le champ `content` contenant le message texte.

## Variables d'environnement utilisees

| Variable | Usage |
|----------|-------|
| `DISCORD_WEBHOOK_URL` | URL du webhook Discord |

## Ajouter un flux RSS

1. Dupliquer un noeud RSS dans n8n
2. Changer l'URL du flux
3. Connecter la sortie au noeud Merge
4. Incrementer `numberInputs` dans le Merge
