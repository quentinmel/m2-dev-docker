# TD4 — Docker : santé, environnements, sécurité et CI/CD

## Partie A — Santé et environnements

### A1.

J'ai ajouté un `healthcheck` aux trois services.

Pour PostgreSQL, j'utilise `pg_isready` :

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
  interval: 5s
  timeout: 5s
  retries: 5
```

Pour Redis, j'utilise `redis-cli ping` :

```yaml
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 5s
  timeout: 5s
  retries: 5
```

Pour l'API, j'utilise `wget`, disponible dans l'image `node:24-alpine`, afin d'effectuer une requête HTTP vers `/health` :

```yaml
healthcheck:
  test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/health"]
  interval: 5s
  timeout: 5s
  retries: 5
  start_period: 5s
```

L'API dépend maintenant de l'état de santé de PostgreSQL et Redis :

```yaml
depends_on:
  db:
    condition: service_healthy
  cache:
    condition: service_healthy
```

J'ai également supprimé `restart: on-failure`, car l'attente des dépendances est maintenant gérée explicitement avec `depends_on` et `condition: service_healthy`.

Pour vérifier le démarrage, j'ai exécuté :

```bash
docker compose down -v
docker compose up -d
docker compose ps
docker compose logs api
```

Les trois services étaient `healthy` :

```text
td4-api-1     Up ... (healthy)
td4-cache-1   Up ... (healthy)
td4-db-1      Up ... (healthy)
```

Les logs de l'API montrent qu'elle s'est connectée correctement à PostgreSQL et Redis et qu'elle a démarré sans crash :

```text
Connecting to Postgres at db:5432…
Connecting to Redis at redis://cache:6379…
Visites API listening on 3000
```

Le `$$` utilisé dans `$${POSTGRES_USER}` permet d'échapper le caractère `$` lors de l'interpolation de Docker Compose. Compose transmet ainsi `$` au conteneur au lieu d'interpréter lui-même la variable. La variable `${POSTGRES_USER}` est ensuite évaluée à l'intérieur du conteneur par le shell exécutant `pg_isready`, à partir de la variable d'environnement `POSTGRES_USER`.

---

### A2.

J'ai séparé la configuration Docker en trois fichiers afin de distinguer la configuration commune, le développement et la production.

Le fichier `compose.yaml` contient la configuration commune aux différents environnements. Il ne publie aucun port et ne contient pas directement le mot de passe PostgreSQL.

Le fichier `compose.override.yaml` est utilisé pour le développement. Il sélectionne la cible `dev` du Dockerfile, active `develop.watch` et publie les ports nécessaires :

```yaml
api:
  ports:
    - "${API_PORT}:3000"

db:
  ports:
    - "${DB_PORT_HOST}:5432"
```

Le fichier `compose.prod.yaml` complète la configuration pour la production. Il utilise la cible `prod`, publie le port de l'API et ajoute `restart: unless-stopped` aux trois services :

```yaml
api:
  ports:
    - "${API_PORT:-8000}:3000"
  restart: unless-stopped

db:
  restart: unless-stopped

cache:
  restart: unless-stopped
```

En production, PostgreSQL et Redis ne possèdent volontairement aucun port publié.

PostgreSQL ne doit pas être directement accessible depuis l'hôte car seule l'API doit communiquer avec la base de données. L'API peut utiliser directement le nom de service `db` sur le réseau Docker. Ne pas publier le port réduit donc la surface d'exposition de la base de données et améliore la sécurité.

J'ai vérifié la configuration finale avec :

```bash
docker compose -f compose.yaml -f compose.prod.yaml config
```

Cette commande permet de vérifier la configuration résultant de la fusion des fichiers Compose. Elle confirme notamment que :

* l'API utilise la cible `prod` ;
* le port de l'API est publié ;
* PostgreSQL et Redis ne publient aucun port ;
* `restart: unless-stopped` est présent sur les trois services ;
* les healthchecks sont conservés.

---

### A3.

J'ai démarré la stack de production avec :

```bash
docker compose -f compose.yaml -f compose.prod.yaml up -d --build
```

J'ai ensuite effectué plusieurs requêtes sur l'API :

```text
{"hitsInRedis":1,"visitsInPostgres":1,"servedBy":"7975e731d1ef"}

{"hitsInRedis":2,"visitsInPostgres":2,"servedBy":"7975e731d1ef"}

{"hitsInRedis":3,"visitsInPostgres":3,"servedBy":"7975e731d1ef"}
```

Les compteurs Redis et PostgreSQL sont donc bien incrémentés à chaque requête.

J'ai ensuite arrêté puis redémarré la stack sans supprimer les volumes :

```bash
docker compose -f compose.yaml -f compose.prod.yaml down

docker compose -f compose.yaml -f compose.prod.yaml up -d
```

Après le redémarrage, une nouvelle requête a retourné :

```text
{"hitsInRedis":4,"visitsInPostgres":4,"servedBy":"ec4c82836087"}
```

Les compteurs ont donc bien conservé leur valeur précédente. Le conteneur API a en revanche changé d'identifiant (`servedBy`), ce qui montre qu'un nouveau conteneur a été créé après le redémarrage.

Les données persistent grâce aux volumes nommés `db-data` et `cache-data`, qui sont conservés lors d'un simple `docker compose down`.

À l'inverse, `docker compose down -v` supprimerait également ces volumes et donc les données persistantes.

La commande `docker compose ps` montre également que seul le port de l'API est publié :

```text
api    0.0.0.0:3000->3000/tcp

cache  6379/tcp

db     5432/tcp
```

PostgreSQL et Redis ne sont donc pas directement accessibles depuis l'hôte. C'est volontaire : la base de données et le cache n'ont pas besoin d'être exposés à l'extérieur de Docker.

Ils communiquent avec l'API via le réseau Docker interne, ce qui réduit la surface d'exposition et améliore la sécurité.

---

### A4.

J'ai mesuré le temps nécessaire pour arrêter uniquement le service API avec :

```bash
time docker compose -f compose.yaml -f compose.prod.yaml stop api
```

Le résultat obtenu est :

```text
[+] Stopping 1/1
 ✔ Container td4-api-1  Stopped  0.4s

real    0m0.706s
user    0m0.103s
sys     0m0.081s
```

L'arrêt complet de la commande a donc pris environ **0,7 seconde**.

J'ai ensuite vérifié l'état du conteneur :

```bash
docker compose -f compose.yaml -f compose.prod.yaml ps -a
```

Le résultat indique :

```text
td4-api-1     Exited (0) 3 seconds ago
td4-cache-1   Up ... (healthy)
td4-db-1      Up ... (healthy)
```

Le code de sortie de l'API est donc **0**, ce qui indique que le processus s'est arrêté proprement.

Le service API possède `restart: unless-stopped`. Cette politique permet au conteneur de redémarrer automatiquement lorsqu'il s'arrête de manière inattendue, par exemple à la suite d'un crash.

En revanche, lorsqu'on utilise volontairement `docker compose stop api`, Docker considère qu'il s'agit d'un arrêt manuel et ne redémarre pas automatiquement le conteneur.

On peut donc distinguer les deux situations :

* **arrêt manuel avec `docker stop` ou `docker compose stop`** → le conteneur reste arrêté ;
* **arrêt inattendu ou crash** → Docker peut redémarrer automatiquement le conteneur grâce à `restart: unless-stopped`.

---

### A5 — Bonus

Pour améliorer la sécurité en production, le mot de passe PostgreSQL peut être fourni à Docker Compose avec un secret plutôt que directement dans les variables d'environnement.

Le secret est placé dans un fichier qui ne doit pas être versionné :

```text
secrets/
└── db_password.txt
```

Le fichier contient uniquement le mot de passe :

```text
td3password
```

Il doit être ajouté au `.gitignore`.

Dans le fichier Compose, le secret peut être déclaré ainsi :

```yaml
secrets:
  db_password:
    file: ./secrets/db_password.txt
```

PostgreSQL peut ensuite utiliser le fichier secret avec :

```yaml
environment:
  POSTGRES_DB: ${POSTGRES_DB}
  POSTGRES_USER: ${POSTGRES_USER}
  POSTGRES_PASSWORD_FILE: /run/secrets/db_password
secrets:
  - db_password
```

L'API peut également lire le mot de passe depuis le fichier secret :

```yaml
environment:
  DB_PASSWORD_FILE: /run/secrets/db_password
secrets:
  - db_password
```

Le fichier `server.js` est prévu pour ce fonctionnement :

```javascript
password: process.env.DB_PASSWORD_FILE
  ? fs.readFileSync(process.env.DB_PASSWORD_FILE, 'utf8').trim()
  : process.env.DB_PASSWORD,
```

Ainsi, le mot de passe est disponible dans le conteneur sous la forme d'un fichier monté dans `/run/secrets/`, plutôt que directement dans la configuration visible des variables d'environnement.

Cette méthode limite donc l'exposition du mot de passe et permet de conserver le secret en dehors du dépôt Git.

---

# Partie B — Sécurité et CI/CD

### B1.

J'ai commencé par utiliser Trivy dans un conteneur Docker afin d'analyser l'image `td4-api` :

```bash
alias trivy='docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache aquasec/trivy:0.74.0'

trivy --version

trivy image td4-api
```

Le scan fonctionne correctement. Trivy a notamment détecté une vulnérabilité de niveau `LOW` concernant `undici` (`CVE-2026-18540`). Les vulnérabilités critiques seront traitées dans la partie B3.

J'ai ensuite testé la transmission d'un secret avec un `ARG` :

```dockerfile
FROM alpine:3.22

ARG TOKEN

RUN echo "Token reçu : $TOKEN"

CMD ["sh"]
```

J'ai construit l'image avec :

```bash
docker build \
  -f secret-test/Dockerfile \
  --build-arg TOKEN="$(cat secret-test/token.txt)" \
  -t td4-secret-arg \
  secret-test
```

Docker a lui-même signalé le problème :

```text
SecretsUsedInArgOrEnv: Do not use ARG or ENV instructions for sensitive data (ARG "TOKEN")
```

La commande suivante a confirmé que le secret était présent dans l'historique de l'image :

```bash
docker history --no-trunc td4-secret-arg
```

On retrouve notamment :

```text
ARG TOKEN=super-secret-td4

RUN |1 TOKEN=super-secret-td4 /bin/sh -c echo "Token reçu : $TOKEN"
```

Cette méthode n'est donc pas adaptée à la gestion de données sensibles.

J'ai ensuite utilisé les secrets BuildKit :

```dockerfile
FROM alpine:3.22

RUN --mount=type=secret,id=token \
    cat /run/secrets/token > /tmp/token-used

CMD ["sh"]
```

L'image a été construite avec :

```bash
docker build \
  --secret id=token,src=secret-test/token.txt \
  -f secret-test/Dockerfile \
  -t td4-secret-buildkit \
  secret-test
```

L'historique obtenu est :

```text
RUN /bin/sh -c cat /run/secrets/token > /tmp/token-used # buildkit
```

Le contenu `super-secret-td4` n'apparaît plus dans l'historique de l'image.

La différence est donc importante : avec `ARG`, le secret peut être enregistré dans les métadonnées et les couches de build. Avec `RUN --mount=type=secret`, le secret est monté temporairement uniquement pendant l'instruction concernée et n'est pas intégré à l'historique de l'image.

Dans une utilisation réelle, il ne faudrait toutefois pas copier le secret vers `/tmp` comme dans cet exemple, car cela le rendrait de nouveau présent dans une couche de l'image. Cette copie sert uniquement à démontrer que le secret était accessible pendant le build.

---

### B2.

J'ai renforcé la sécurité du conteneur API en production dans `compose.prod.yaml`.

J'ai ajouté les paramètres suivants :

```yaml
read_only: true

tmpfs:
  - /tmp

cap_drop:
  - ALL

mem_limit: 256m
```

L'option `read_only: true` rend le système de fichiers du conteneur accessible uniquement en lecture. Cela limite les possibilités de modification du conteneur en cas de compromission de l'application.

Le répertoire `/tmp` reste cependant accessible en écriture grâce à `tmpfs`. Les fichiers qui y sont créés sont stockés temporairement et ne sont pas conservés après la suppression du conteneur.

J'ai vérifié que l'application fonctionnait toujours normalement après cette modification avec :

```bash
curl http://localhost:3000/
```

J'ai ensuite vérifié que l'écriture dans `/tmp` était possible :

```bash
docker compose -f compose.yaml -f compose.prod.yaml exec api \
  sh -c 'echo test > /tmp/test.txt && cat /tmp/test.txt'
```

L'écriture dans `/tmp` fonctionne correctement.

À l'inverse, une tentative d'écriture dans `/app` échoue car le système de fichiers est en lecture seule :

```bash
docker compose -f compose.yaml -f compose.prod.yaml exec api \
  sh -c 'echo test > /app/test.txt'
```

Cette restriction est souhaitable : l'application ne doit pas pouvoir modifier librement son propre système de fichiers.

J'ai également supprimé toutes les Linux capabilities avec :

```yaml
cap_drop:
  - ALL
```

La vérification de `CapEff` dans `/proc/1/status` confirme que les capabilities effectives ont été supprimées.

Enfin, j'ai limité la mémoire du conteneur à 256 Mo :

```yaml
mem_limit: 256m
```

La valeur configurée dans Docker correspond à **268 435 456 octets**, soit 256 MiB.

Le conteneur API dispose donc maintenant de plusieurs protections complémentaires : système de fichiers en lecture seule, répertoire temporaire isolé et inscriptible, suppression des capabilities Linux et limitation de la mémoire.

Les fichiers nécessaires au fonctionnement permanent de l'application ne doivent pas être stockés dans le système de fichiers du conteneur. Les données persistantes doivent être placées dans une base de données, un volume Docker ou un stockage externe adapté.

Dans notre architecture, les données de l'application sont notamment conservées dans PostgreSQL et Redis via leurs volumes dédiés.

---

### B3.

J'ai créé une image de test contenant volontairement une version vulnérable de `lodash` :

```dockerfile
FROM node:24-alpine

WORKDIR /app

RUN npm install lodash@4.17.4

CMD ["node", "--version"]
```

Après construction de l'image, j'ai lancé un scan Trivy en demandant un code d'erreur lorsqu'une vulnérabilité `CRITICAL` est détectée :

```bash
trivy image --severity CRITICAL --exit-code 1 td4-b3-vulnerable
```

Trivy a détecté une vulnérabilité critique :

```text
Total: 1 (CRITICAL: 1)

lodash  4.17.4

CVE-2019-10744

CRITICAL

Fixed Version: 4.17.12
```

Il s'agit d'une vulnérabilité de type *prototype pollution* dans `lodash`.

Le code de sortie obtenu était :

```text
1
```

Le pipeline de sécurité peut donc être configuré pour échouer automatiquement lorsqu'une vulnérabilité critique est présente.

J'ai ensuite corrigé l'image en utilisant la version `4.17.12` de `lodash`, indiquée par Trivy comme version corrigée :

```dockerfile
FROM node:24-alpine

WORKDIR /app

RUN npm install lodash@4.17.12

CMD ["node", "--version"]
```

Après reconstruction, j'ai relancé le même scan :

```bash
trivy image --severity CRITICAL --exit-code 1 td4-b3-fixed
```

Le code de sortie obtenu était :

```text
0
```

Il n'y a donc plus de vulnérabilité `CRITICAL` détectée dans l'image corrigée.

Cette vérification montre qu'un scan Trivy avec `--severity CRITICAL --exit-code 1` peut être utilisé comme garde-fou dans une chaîne CI/CD : une image contenant une vulnérabilité critique fait échouer l'étape de sécurité, tandis qu'une image corrigée permet au pipeline de continuer.

---

### B4.

Le projet étant hébergé sur GitHub, j'ai utilisé GitHub Actions pour mettre en place la chaîne CI/CD.

Le workflow se trouve dans :

```text
.github/workflows/docker.yml
```

Le pipeline est composé de quatre jobs :

```text
build
  ↓
scan
  ↓
test
  ↓
publish
```

#### Build

Le premier job récupère le dépôt et construit l'image Docker à partir du Dockerfile de l'application :

```bash
docker build -t "$IMAGE_NAME" ./td4/app
```

L'image est ensuite exportée dans une archive :

```bash
docker save "$IMAGE_NAME" -o image.tar
```

Cette archive est transférée au job suivant avec un artifact GitHub Actions.

#### Scan

Le job `scan` récupère l'artifact et recharge l'image :

```bash
docker load -i image.tar
```

Trivy est ensuite exécuté dans un conteneur Docker :

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v trivy-cache:/root/.cache \
  aquasec/trivy:0.74.0 \
  image \
  --severity CRITICAL \
  --exit-code 1 \
  "$IMAGE_NAME"
```

L'option `--exit-code 1` permet de faire échouer le job si une vulnérabilité `CRITICAL` est détectée.

#### Test

Le job `test` recharge également l'image depuis l'artifact :

```bash
docker load -i image.tar
```

Il exécute ensuite l'image avec `docker run` :

```bash
docker run --rm "$IMAGE_NAME" node --version
```

Le test utilise uniquement l'image Docker et ne monte aucun fichier du dépôt dans le conteneur. Le conteneur Docker n'a donc pas accès directement aux fichiers du dépôt.

#### Publication

Le dernier job publie l'image dans GitHub Container Registry (GHCR).

Il dépend du succès du job `test` :

```yaml
needs: test
```

La publication est également limitée aux push sur la branche `main` :

```yaml
if: github.event_name == 'push' && github.ref == 'refs/heads/main'
```

Le workflow utilise le token fourni automatiquement par GitHub :

```yaml
permissions:
  contents: read
  packages: write
```

La connexion à GHCR est réalisée avec :

```yaml
registry: ghcr.io
username: ${{ github.actor }}
password: ${{ secrets.GITHUB_TOKEN }}
```

L'image est ensuite publiée avec :

```bash
docker push "$IMAGE_NAME"
```

Le nom de l'image utilise le SHA du commit :

```yaml
IMAGE_NAME: ghcr.io/${{ github.repository }}/visites-api:${{ github.sha }}
```

Cela permet d'identifier précisément la version de l'image correspondant au commit utilisé pour sa construction.

Le fonctionnement attendu est donc le suivant :

```text
Pull Request :
Build → Scan → Test
                  ↓
              pas de publication

Push sur main :
Build → Scan → Test → Publish GHCR
```

Si le scan Trivy détecte une vulnérabilité `CRITICAL`, le job `scan` échoue. Les jobs suivants ne sont alors pas exécutés et l'image n'est pas publiée.

De la même manière, si le test échoue, le job `publish` ne peut pas s'exécuter.

L'image est transférée entre les jobs avec `docker save`, un artifact GitHub Actions et `docker load`. Cela permet aux différents jobs d'utiliser exactement la même image sans reconstruire l'image à chaque étape.

---

### B5.

Pour utiliser l'image publiée dans GHCR sur une machine de déploiement, il est possible de fournir son nom à Compose avec la variable `API_IMAGE`.

Le fichier `compose.prod.yaml` utilise cette variable :

```yaml
image: ${API_IMAGE:-td4-api}
```

La valeur par défaut est `td4-api`, mais elle peut être remplacée par une image publiée dans GHCR.

Après s'être connecté au registre :

```bash
docker login ghcr.io
```

on peut lancer la stack de production avec :

```bash
API_IMAGE=ghcr.io/<owner>/<repository>/visites-api:<commit> \
docker compose \
  -f compose.yaml \
  -f compose.prod.yaml \
  up -d --no-build
```

`--no-build` est important car il empêche Docker de reconstruire localement l'image. La machine de déploiement utilise donc directement l'image déjà construite et publiée par le pipeline CI/CD.

Le tag correspondant au SHA du commit permet d'identifier précisément la version déployée. Contrairement à un tag générique comme `latest`, un tag basé sur le commit permet de savoir exactement quel code source a produit l'image.

Le principe est donc :

```text
Code source
    ↓
GitHub Actions
    ↓
Build Docker
    ↓
Scan Trivy
    ↓
Tests
    ↓
GHCR
    ↓
docker compose --no-build
    ↓
Image correspondant au commit
```

Cette approche permet de séparer clairement la construction de l'image et son déploiement. L'image déployée correspond directement à celle qui a été construite, analysée et validée par la CI.
