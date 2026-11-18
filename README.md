# quantique.romainmurier.com (VPS Airflow)

Site de la présentation « Introduction à l'informatique quantique » (Forum Technique 2026).

- `site/index.html` : les sources, à l'adresse https://quantique.romainmurier.com
- `site/jeu/index.html` : le jeu, à l'adresse https://quantique.romainmurier.com/jeu/

## Principe

Sur le VPS Airflow, Traefik v3.3 (conteneur `traefik`) utilise uniquement le **provider fichier** (`/dynamic.yml`, rechargé à chaud). Les entrypoints sont `web` (80, redirigé globalement vers HTTPS) et `websecure` (443), et le certresolver s'appelle `letsencrypt` (challenge HTTP).

- Un conteneur `nginx:1.27-alpine` nommé `quantique` sert `/opt/quantique/site`, sur le réseau `traefik-net`, sans port publié.
- La route `quantique.romainmurier.com` vers `http://quantique:80` est déclarée une fois pour toutes dans le fichier dynamique de Traefik (`traefik/quantique.yml`).
- À chaque push sur `main` qui touche `site/`, GitHub Actions copie le dossier avec rsync. Ni le conteneur ni Traefik ne sont redémarrés.

## Mise en place (une seule fois)

### 1. Conteneur nginx

Depuis le dossier du dépôt, sur ton poste :

```bash
ssh -t debian@HOST 'sudo apt install -y rsync && sudo mkdir -p /opt/quantique/site && sudo chown -R debian /opt/quantique'
rsync -az docker-compose.yml site debian@HOST:/opt/quantique/
ssh debian@HOST 'cd /opt/quantique && docker compose up -d && docker ps --filter name=quantique'
```

### 2. Route Traefik

Sur le VPS :

```bash
docker inspect traefik --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
DYN=/chemin/affiché/pour/dynamic.yml
sudo cp "$DYN" "$DYN.bak-$(date +%F)"
sudo nano "$DYN"
```

Ajouter le routeur `quantique` sous `http.routers` et le service `quantique` sous `http.services` (voir `traefik/quantique.yml`), sans dupliquer les clés `http:`, `routers:` ou `services:`.

Puis vérifier que Traefik a rechargé la configuration :

```bash
docker logs traefik --since 5m 2>&1 | tail -20
curl -I https://quantique.romainmurier.com
```

Si rien ne change au bout de 30 secondes, Traefik ne voit pas la nouvelle version du fichier (fréquent avec un fichier monté seul, quand l'éditeur le remplace au lieu de le réécrire). Dans ce cas : `docker restart traefik`, ce qui coupe Airflow et SonarQube quelques secondes.

### 3. Secrets GitHub

Settings > Secrets and variables > Actions :

| Secret | Valeur |
| --- | --- |
| `SERVER_HOST` | le même que pour les dépôts Airflow |
| `SERVER_USER` | `debian` (doit posséder `/opt/quantique`) |
| `SSH_KEY` | recommandé : une clé privée dédiée au déploiement |
| `SERVER_PASSWORD` | sinon : le mot de passe utilisé par les dépôts Airflow |
| `SERVER_PORT` | seulement si SSH n'écoute pas sur le port 22 |

## Mettre à jour le site

Modifier un fichier dans `site/`, commit, push sur `main`. Le site est à jour environ 20 secondes plus tard.

## En cas de problème

- **404 page not found** : la route n'est pas chargée (erreur de YAML dans le fichier dynamique, ou fichier non rechargé). Regarder `docker logs traefik`.
- **Bad Gateway** : le conteneur `quantique` n'est pas démarré ou n'est pas sur `traefik-net` (`docker network inspect traefik-net`).
- **Certificat invalide** : regarder `docker logs traefik 2>&1 | grep -i acme` ; le plus souvent, le DNS n'est pas encore propagé.
- **Le workflow échoue sur rsync** : `rsync` absent du serveur, ou `debian` ne possède pas `/opt/quantique/site`.
