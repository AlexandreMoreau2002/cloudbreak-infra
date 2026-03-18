# CLAUDE.md — Infra

## Workflow de développement

Après chaque modification des fichiers Docker Compose ou Caddyfile :

```bash
# Vérifier la syntaxe Docker Compose
docker compose -f docker-compose.dev.yml config   # dev
docker compose -f docker-compose.yml config       # prod

# Relancer l'infra locale après modification
docker compose -f docker-compose.dev.yml down
docker compose -f docker-compose.dev.yml up -d

# Vérifier que les containers tournent
docker compose -f docker-compose.dev.yml ps
docker compose -f docker-compose.dev.yml logs
```

**Règles :**
- Ne jamais commiter des secrets dans les fichiers Docker Compose — utiliser des variables d'environnement
- `docker-compose.dev.yml` : db + redis uniquement (pas de backend, pas de caddy)
- `docker-compose.yml` : stack complète prod (api + db + redis + caddy)
- Tester que le backend démarre correctement après toute modification infra
