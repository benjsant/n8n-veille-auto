# n8n Veille Auto

Veille data engineering automatisee avec n8n : RSS, deduplication, digest Discord, et rapport IA quotidien via DeepSeek.

## Workflows disponibles

Deux workflows independants dans `workflows/` :

- **veille-rss-discord-medium-limit.json** : Lit 4 flux RSS toutes les 24h, filtre les articles recents, deduplique, et envoie un digest sur Discord. Limite a 3 articles Medium max pour eviter le spam.

- **phase2-deepseek-discord-embeds-v3-prompt-improved.json** : Appelle DeepSeek pour generer un rapport de veille structure (AI, Data Engineering, MLOps, Cloud). Formate en embeds Discord avec scores et recommandations.

Le dossier `workflows/Nouveau dossier/` contient des versions anterieures gardees comme reference.

## Installation

```bash
git clone <url-du-repo>
cd n8n-veille-auto
cp .env.example .env
# Remplir .env avec vos valeurs
docker compose up -d
```

n8n tourne sur `http://localhost:5678`. Importez les workflows via le menu Import from File.

## Configuration .env

| Variable | Description |
|----------|-------------|
| `TZ` | Fuseau horaire (ex: `Europe/Paris`) |
| `DISCORD_WEBHOOK_URL` | Webhook pour les digests RSS |
| `DISCORD_WEBHOOK_URL_AI` | Webhook pour les rapports IA |
| `DEEPSEEK_API_KEY` | Cle API DeepSeek |

## Flux RSS configures

- Blog OCTO (architecture, craft)
- Towards Data Science (ML)
- Medium tag data-engineering
- Practical Data Engineering (Substack)

Pour ajouter un flux : dupliquer un noeud RSS dans n8n, changer l'URL, connecter au Merge et incrementer `numberInputs`.

## Deduplication

Les workflows RSS stockent les liens deja envoyes via `$getWorkflowStaticData('global')`. Les entrees de plus de 30 jours sont nettoyees automatiquement.

## Licence

MIT
