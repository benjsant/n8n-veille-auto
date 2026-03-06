# Workflow DeepSeek -> Discord Embeds

**Fichier** : `workflows/phase2-deepseek-discord-embeds-v3-prompt-improved.json`

## Objectif

Generer un rapport de veille IA quotidien via l'API DeepSeek et le poster sur Discord sous forme d'embeds structures et colores.

## Schema du workflow

```
Schedule (8h)
      |
      v
 DeepSeek LLM (HTTP Request)
      |
      v
 Parse + Build Embeds (Code JS)
      |
      v
 Discord Webhook (HTTP Request)
```

## Detail des noeuds

### Schedule

Declenchement tous les jours a 8h du matin.

### DeepSeek LLM (noeud HTTP Request)

Appel POST vers `https://api.deepseek.com/chat/completions`.

**Configuration :**
- Modele : `deepseek-chat`
- Temperature : 0 (reponses deterministes, pas de creativite)
- Max tokens : 1500
- Timeout : 60 secondes
- `continueOnFail: true` (le workflow continue meme si l'API plante)

**Authentification :**
Header `Authorization: Bearer $env.DEEPSEEK_API_KEY`

**Prompt system :**
Role d'analyste senior en veille techno. Consignes : repondre en francais, renvoyer uniquement du JSON parseable, pas de markdown.

**Prompt user :**
Demande un rapport structure avec :
- Un titre (max 60 chars)
- Une date (DD/MM/YYYY)
- 4 sections dans cet ordre : AI, Data Engineering, MLOps, Cloud
- Chaque section contient :
  - `name` : nom exact de la section
  - `importance` : high, medium ou low
  - `score` : entier 0-100 (impact strategique)
  - `bullets` : exactement 3 tendances (max 110 chars chacune)
  - `why_it_matters` : 1 phrase explicative (max 180 chars)
- 3 actions recommandees (`top_actions`)

Le prompt demande de decrire des tendances verifiables plutot qu'inventer des annonces specifiques.

**Schema JSON attendu en sortie :**
```json
{
  "title": "...",
  "date": "DD/MM/YYYY",
  "sections": [
    {
      "name": "AI",
      "importance": "high",
      "score": 85,
      "bullets": ["...", "...", "..."],
      "why_it_matters": "..."
    }
  ],
  "top_actions": ["...", "...", "..."]
}
```

### Parse + Build Embeds (noeud Code JS)

Gestion d'erreurs en cascade, puis construction des embeds Discord.

**Etape 1 : Validation (4 cas d'erreur)**

| Cas | Condition | Resultat |
|-----|-----------|----------|
| Erreur API | `raw.error` existe | Embed rouge avec message d'erreur |
| Structure inattendue | Pas de `choices[0].message` | Embed orange avec extrait brut |
| Contenu vide | `content` est vide | Embed rouge |
| JSON invalide | `JSON.parse()` echoue | Embed orange avec le contenu brut |

A chaque cas d'erreur, le workflow s'arrete la et envoie l'embed d'erreur sur Discord (pas de crash silencieux).

**Etape 2 : Construction des embeds (si tout va bien)**

| Embed | Contenu | Couleur |
|-------|---------|---------|
| Header | Titre du rapport + date | Bleu (5793266) |
| Section high | Nom + score + bullets + why | Orange (15105570) |
| Section medium | Nom + score + bullets + why | Bleu (3447003) |
| Section low | Nom + score + bullets + why | Vert (2067276) |
| Actions | 3 recommandations | Vert (2067276) |

Format du titre de section : `AI [emoji] HIGH (85/100)`

Emojis par importance : high = feu, medium = etoile, low = info

Toutes les strings sont tronquees aux limites definies (titres 200 chars, bullets 120, why 180, actions 140). Maximum 10 embeds au total (limite Discord).

### Discord Webhook

POST HTTP sur `$env.DISCORD_WEBHOOK_URL_AI` avec le payload :
```json
{
  "embeds": [...]
}
```

## Variables d'environnement utilisees

| Variable | Usage |
|----------|-------|
| `DEEPSEEK_API_KEY` | Cle API DeepSeek |
| `DISCORD_WEBHOOK_URL_AI` | URL du webhook Discord dedie aux rapports IA |

## Limites connues

- DeepSeek peut halluciner des tendances qui n'existent pas (le prompt essaie de limiter ca)
- Le rapport n'est pas base sur des sources reelles, contrairement au workflow RSS
- Si l'API DeepSeek est down ou lente (>60s), un embed d'erreur est envoye a la place
- Le score d'importance est subjectif (genere par le LLM)
