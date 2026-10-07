# TD4 — Vers la production (≈ 2 h 30, en deux parties)

**Individuel.** Rendu dans votre dépôt `docker-td`, avant la **date donnée en cours** :
```
td4/                       ← partie A (copie de votre td3/ + ce qui suit)
├── app/
├── compose.yaml
├── compose.override.yaml
├── compose.prod.yaml
├── .env.example
└── REPONSES.md            ← réponses A et B, avec commandes et sorties
ET à la racine du dépôt :
.gitlab-ci.yml             ← partie B (ou .github/workflows/td4.yml si votre dépôt est sur GitHub)
```

> Commandes écrites pour un terminal **bash / zsh** (macOS, Linux, ou Ubuntu dans WSL sous Windows — recommandé).
> Sous PowerShell, remplacez `time <commande>` par `Measure-Command { <commande> }`.

---

## Partie A — L'app du TD3 en « prod » (matin, ≈ 1 h)

Copiez votre `td3/` (app comprise) dans `td4/`.

**A1. Démarrage fiable.** Ajoutez un `healthcheck` à `db` (`pg_isready`), à `cache` (`redis-cli ping`) et à `api`
(`GET /health` — quel outil est disponible dans l'image `node:24-alpine` pour faire une requête HTTP ?).
L'API attend que `db` et `cache` soient **sains**. Retirez `restart: on-failure` s'il ne servait qu'à ça.

> Prouvez qu'il n'y a plus de crash au démarrage (`docker compose down -v && docker compose up -d`, puis les logs).
> À quoi sert `$$` dans `pg_isready -U $${POSTGRES_USER}` ?

**A2. Trois fichiers.** Restructurez :
- `compose.yaml` : la base commune — **aucun port publié**, **aucun mot de passe** ;
- `compose.override.yaml` : le dev (cible `dev`, `develop.watch`, port de l'API, port de la base, mot de passe lu dans `.env`) ;
- `compose.prod.yaml` : l'API publiée sur le port `${API_PORT:-8000}`, `restart: unless-stopped` sur chaque service,
  la base **non** publiée, et le mot de passe (lu dans `.env` — ou en secret, cf. A5).

Pourquoi le mot de passe n'est-il pas dans la base ? Parce qu'il ne vient **pas du même endroit** en dev et en prod :
chaque environnement apporte le sien.

> Pourquoi la base ne doit-elle publier **aucun** port ?
> Vérifiez la fusion avec `docker compose -f compose.yaml -f compose.prod.yaml config`.

**A3. La prod.** Lancez-la :
```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d --build   # --build : sinon l'image de la cible dev est réutilisée
```
Appelez `GET /` plusieurs fois, puis `docker compose -f compose.yaml -f compose.prod.yaml down` et `up -d` à nouveau.

> Les compteurs ont-ils survécu ? Pourquoi ? La base est-elle joignable depuis votre machine ? Pourquoi est-ce voulu ?

**A4. Un arrêt propre.** Arrêtez l'API en chronométrant (`time docker compose … stop api`), puis relevez son code de
sortie (`docker compose … ps -a`).

> Combien de temps ? Quel code de sortie, et qu'est-ce qu'il signifie ? Que fait `restart: unless-stopped`
> après un `docker stop` ? Et si le conteneur plante ?

**A5. (bonus) Le mot de passe en secret.** En prod, le mot de passe ne passe plus par une variable d'environnement :
fichier `secrets/db_password.txt` (hors git), déclaré en `secrets:`, lu par Postgres via `POSTGRES_PASSWORD_FILE`
et par l'API via `DB_PASSWORD_FILE`.

> Comparez la sortie de `docker inspect` sur l'API avant / après.

---

## Partie B — Sécuriser et livrer (après-midi, ≈ 1 h 30)

On continue dans `td4/`.

Trivy s'utilise **en conteneur** (rien à installer) — copiez cette ligne dans votre terminal :

```bash
alias trivy='docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache aquasec/trivy:0.74.0'
```

**B1. Un secret de build qui ne fuit pas.** L'équipe a besoin, pendant le build, d'un jeton privé (fichier
`token.txt`, **hors git**). Simulez-le en ajoutant au début de la cible `prod` une instruction qui affiche le jeton
(`RUN echo "téléchargement avec le jeton …"`).

1. Faites-le d'abord avec un `ARG TOKEN` passé par `--build-arg`. Retrouvez la valeur du jeton **dans l'image**.
2. Refaites-le avec `RUN --mount=type=secret,id=token` et `docker build --secret id=token,src=token.txt`.
   Prouvez que le jeton n'est plus dans l'image.

> Quelle commande vous a permis de retrouver le jeton ? Pourquoi le secret monté, lui, ne laisse-t-il pas de trace ?

**B2. Durcir l'API.** Dans `compose.prod.yaml`, l'API tourne en lecture seule (sauf `/tmp`), sans aucune
*capability* (`cap_drop: [ALL]`) et avec 256 Mo de mémoire au maximum.

> Prouvez que l'application fonctionne toujours, puis que chaque protection est active :
> une écriture refusée, `grep CapEff /proc/1/status` dans le conteneur, `docker inspect` pour les limites.
> Pourquoi faut-il laisser `/tmp` inscriptible ? Et où iraient des fichiers que l'app doit **garder** ?

**B3. Un scan qui bloque.**
```bash
trivy image --severity CRITICAL --exit-code 1 <votre-image> ; echo "code de sortie : $?"
```
> Quel code de sortie ? Ajoutez temporairement une dépendance vulnérable (Node : `npm install lodash@4.17.4`),
> reconstruisez et relancez le scan : que se passe-t-il ? Retirez-la ensuite.

**B4. La pipeline.** Écrivez une pipeline **build → scan → test → publish** :

| Étape | Ce qu'elle fait | Quand |
|---|---|---|
| build | construit l'image (cible `prod`) | toujours |
| scan | Trivy, **échoue** s'il reste une faille CRITICAL | toujours |
| test | lance l'image avec Postgres et Redis, et **attend qu'elle réponde** à `GET /` | toujours |
| publish | pousse l'image dans le registry, tag = le commit | branche par défaut |

Dans `.gitlab-ci.yml`, sur GitLab :
- chaque job a son propre Docker (service `docker:29-dind`) : l'image passe d'un job à l'autre en fichier
  (`docker save` / `docker load`, déclaré en `artifacts`) ;
- ce Docker ne voit **pas** les fichiers de votre dépôt : pour le test, des `docker run` (réseau, variables
  d'environnement), sans montage de fichier ;
- pour publier : `docker login` avec `$CI_REGISTRY_USER` / `$CI_REGISTRY_PASSWORD` (fournis par GitLab), image
  `$CI_REGISTRY_IMAGE/visites-api:$CI_COMMIT_SHORT_SHA`.

*Dépôt de TD sur GitHub ?* Même pipeline en GitHub Actions : un seul job suffit (Docker est déjà sur la machine),
publication sur GHCR avec `GITHUB_TOKEN`.

> Montrez une pipeline **rouge** (une dépendance vulnérable ajoutée exprès, ou un test qui échoue) : l'image
> est-elle publiée ? Puis la pipeline **verte**.

**B5. « Déployer ».** Sur votre machine, lancez la prod avec l'image publiée par la pipeline :
```bash
docker login <registry>                                  # dépôt privé : identifiant + jeton personnel
API_IMAGE=<registry>/<chemin>/visites-api:<commit> docker compose -f compose.yaml -f compose.prod.yaml up -d --no-build
```
(dans `compose.prod.yaml`, le service `api` a `image: ${API_IMAGE:-td4-api}`).

> Pourquoi `--no-build` ? Pourquoi un tag lié au commit plutôt que `latest` ?

---

### Barème (/10)

| Critère | Points |
|---|---|
| A1 healthchecks + `service_healthy`, démarrage sans crash prouvé | 1,5 |
| A2 trois fichiers corrects (base sans port ni mot de passe) | 1,5 |
| A3-A4 : preuves et explications | 2 |
| B1 : fuite prouvée puis corrigée avec un secret de build | 1 |
| B2 : API durcie, protections prouvées, app fonctionnelle | 1 |
| B3 : scan bloquant, réponses | 0,5 |
| B4 : pipeline build → scan → test → publish, rouge puis verte | 2 |
| B5 : prod lancée depuis l'image publiée, réponses | 0,5 |
| A5 (bonus) | +0,5 |
