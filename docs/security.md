# Security — cloudbreak-infra

Document de référence sécurité pour l'infrastructure. À mettre à jour à chaque changement réseau, firewall, ou config Docker.

---

## État actuel (story 1-2 — docker-compose)

### Ports exposés — Dev local

| Port | Service | Accessible depuis |
|------|---------|-------------------|
| 8001 | FastAPI | localhost uniquement |
| 5432 | PostgreSQL | localhost uniquement |
| 6379 | Redis | localhost uniquement |

### Ports exposés — Prod (VPS)

| Port | Service | Accessible depuis |
|------|---------|-------------------|
| 80 | Caddy (redirect HTTPS) | Internet |
| 443 | Caddy (HTTPS) | Internet |
| 8000 | FastAPI | Réseau Docker interne uniquement |
| 5432 | PostgreSQL | Réseau Docker interne uniquement |
| 6379 | Redis | Réseau Docker interne uniquement |

> **Règle** : en prod, PostgreSQL et Redis ne sont **jamais** mappés sur un port host. Seul Caddy est exposé à internet.

---

## Réseau Docker

Les services communiquent via le réseau bridge Docker interne (`infra_default`). Ce réseau est isolé — aucun accès depuis l'extérieur sans mapping de port explicite.

```yaml
# Correct — pas de ports sur db et redis en prod
db:
  image: postgres:16-alpine
  # pas de ports: mapping ici

redis:
  image: redis:7-alpine
  # pas de ports: mapping ici
```

---

## Firewall VPS (à configurer à l'installation)

```bash
# UFW — seuls ports autorisés
ufw allow 22    # SSH
ufw allow 80    # HTTP (redirect Caddy)
ufw allow 443   # HTTPS (Caddy)
ufw enable
```

Tout le reste est bloqué par défaut. **Ne jamais ouvrir 5432 ou 6379 sur le VPS.**

---

## Caddy & TLS

- Certificats Let's Encrypt générés et renouvelés automatiquement par Caddy
- TLS 1.2 minimum (Caddy par défaut)
- HTTPS forcé — Caddy redirige HTTP → HTTPS automatiquement
- Certificats stockés dans le volume `caddy_data` (persistant entre les restarts)

---

## Accès SSH VPS

```bash
# Désactiver l'auth par mot de passe (à faire après avoir copié ta clé SSH)
# Dans /etc/ssh/sshd_config :
PasswordAuthentication no
PermitRootLogin no
```

Accès uniquement par clé SSH depuis ta machine. La clé privée ne quitte jamais ton Mac.

---

## Données persistantes

| Volume | Contenu | Backup requis |
|--------|---------|---------------|
| `pgdata` | Base de données PostgreSQL | ✅ Oui — avant chaque migration |
| `caddy_data` | Certificats TLS | ⚠️ Optionnel (Let's Encrypt les re-génère) |
| `caddy_config` | Config Caddy runtime | ❌ Non |

### Backup PostgreSQL

```bash
# Dump manuel
docker exec infra-db-1 pg_dump -U postgres cloudbreak > backup_$(date +%Y%m%d).sql

# Restore
cat backup.sql | docker exec -i infra-db-1 psql -U postgres cloudbreak
```

---

## Checklist sécurité infra (avant prod)

- [ ] Firewall UFW configuré (22, 80, 443 uniquement)
- [ ] SSH par clé uniquement — `PasswordAuthentication no`
- [ ] `PermitRootLogin no` dans sshd_config
- [ ] PostgreSQL et Redis sans port mappé dans `docker-compose.yml`
- [ ] Caddy HTTPS opérationnel — vérifier avec `curl https://api.cloudbreak.app/health`
- [ ] Volumes Docker persistants vérifiés (`docker volume ls`)
- [ ] Backup PostgreSQL automatisé (cron ou script)
- [ ] Variables `.env` sur le VPS — permissions `chmod 600 .env`
