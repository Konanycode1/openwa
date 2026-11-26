# Déploiement en production — openwa.abrahamkonany.com

## Fonctionnement

1. Un push sur `main` lance le workflow **CI** (lint, tests, build). Il construit l'image Docker,
   vérifie qu'elle démarre, puis la publie sur `ghcr.io/konanycode1/openwa:<sha-court>`.
2. Si la CI réussit, le workflow **Deploy production** (`.github/workflows/deploy.yml`) se connecte
   au VPS en SSH, copie les fichiers compose dans `/home/konan/apps/openwa` et télécharge l'image testée. Il
   redémarre ensuite la stack : API et Postgres.
3. Le conteneur nginx déjà présent sur le serveur (`~/infra/nginx`) gère le HTTPS. Il joint l'API
   par le réseau Docker partagé `networkings` (`http://openwa-api:2785`) et sert l'API, le
   dashboard et les WebSockets sur `https://openwa.abrahamkonany.com`.

Pour un rollback : **Actions → Deploy production → Run workflow**, avec le SHA court d'un commit
déjà construit par la CI.

## Mise en place (une seule fois)

### 1. DNS

Créez un enregistrement `A` (et `AAAA` si vous avez de l'IPv6) qui fait pointer
`openwa.abrahamkonany.com` vers l'IP du VPS. Faites-le avant l'étape nginx : certbot en a besoin
pour obtenir le certificat.

### 2. VPS

Prérequis : Docker Engine, le plugin `docker compose`, le conteneur `nginx` de `~/infra/nginx`
et le réseau Docker externe `networkings`.

```bash
# L'utilisateur konan doit pouvoir lancer docker sans sudo (se reconnecter ensuite)
sudo usermod -aG docker konan
mkdir -p ~/apps/openwa
```

### 3. Clé SSH de déploiement

Sur votre machine :

```bash
ssh-keygen -t ed25519 -N "" -C "github-actions-openwa" -f ~/.ssh/openwa-deploy
ssh-copy-id -i ~/.ssh/openwa-deploy.pub konan@<IP_DU_VPS>
ssh-keyscan -H <IP_DU_VPS>          # sortie à mettre dans le secret VPS_KNOWN_HOSTS
```

### 4. Fichier `.env` de production

Copiez `deploy/.env.production.example` vers `/home/konan/apps/openwa/.env` sur le serveur, puis remplacez
chaque valeur `<...>` (`openssl rand -hex 32`). Protégez ensuite le fichier :

```bash
chmod 600 ~/apps/openwa/.env
```

`API_KEY_PEPPER` ne doit plus jamais changer après le premier démarrage, sinon toutes les clés API
deviennent invalides. Conservez ces valeurs dans un gestionnaire de mots de passe.

### 5. Secrets GitHub

Dans **Settings → Environments**, créez l'environnement `production`. Ajoutez-y ces secrets :

| Secret            | Valeur                                                |
| ----------------- | ----------------------------------------------------- |
| `VPS_HOST`        | IP ou nom d'hôte du VPS                               |
| `VPS_USER`        | `konan`                                               |
| `VPS_SSH_KEY`     | contenu de `~/.ssh/openwa-deploy` (la clé **privée**) |
| `VPS_KNOWN_HOSTS` | sortie de `ssh-keyscan -H <IP_DU_VPS>`                |
| `VPS_PORT`        | facultatif, port SSH s'il est différent de 22         |

Vous pouvez aussi ajouter une règle d'approbation manuelle sur cet environnement : chaque
déploiement attendra alors votre validation.

### 6. nginx et certificat HTTPS

Le modèle de configuration est dans [`deploy/nginx/openwa.conf`](nginx/openwa.conf). Ajoutez-le en
deux temps au fichier `~/infra/nginx/conf.d/nginx.conf`, en testant avant chaque rechargement, car
ce nginx sert aussi les autres sites :

1. Ajoutez d'abord **uniquement le bloc HTTP** (port 80), puis testez et rechargez :
   `docker exec nginx nginx -t && docker exec nginx nginx -s reload`.
2. Obtenez le certificat :

   ```bash
   docker run --rm \
     -v /home/konan/infra/nginx/certbot/conf:/etc/letsencrypt \
     -v /home/konan/infra/nginx/certbot/www:/var/www/certbot \
     certbot/certbot certonly --webroot -w /var/www/certbot \
     -d openwa.abrahamkonany.com --non-interactive --agree-tos -m <votre-email>
   ```

3. Ajoutez le **bloc HTTPS** (port 443), puis testez et rechargez de la même façon.

Le renouvellement est déjà assuré par la tâche cron `certbot renew` du serveur.

### 7. Premier déploiement et clé API

Après le premier déploiement réussi, récupérez la clé admin sur le serveur :

```bash
docker exec openwa-api cat /app/data/.api-key
```

## Commandes utiles sur le VPS

```bash
cd ~/apps/openwa
cat .deployed-image                                  # image actuellement en production
docker compose -f docker-compose.yml -f docker-compose.prod.yml --profile postgres ps
docker logs -f openwa-api
docker logs nginx                                    # reverse proxy / HTTPS
```

## Sauvegardes

Il faut sauvegarder **deux** volumes Docker :

- `openwa_postgres-data` : la base de données des messages ;
- `openwa_openwa-data` : les sessions WhatsApp, et une base SQLite (`main.sqlite`) qui contient les
  clés API et le journal d'audit, même quand Postgres est utilisé.

```bash
docker exec openwa-postgres pg_dump -U openwa openwa | gzip > openwa-$(date +%F).sql.gz
docker run --rm -v openwa_openwa-data:/data:ro -v "$PWD":/backup alpine \
  tar czf /backup/openwa-data-$(date +%F).tar.gz -C /data .
```

Le projet fournit aussi `scripts/backup.sh` et `scripts/restore.sh` (voir
`docs/11-operational-runbooks.md`).
