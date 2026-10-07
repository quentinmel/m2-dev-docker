****A1.****

Premier essai sans réseau :

L'API ne démarre pas et s'arrête avec l'erreur `getaddrinfo ENOTFOUND td3-db`.

L'API essaie de contacter PostgreSQL avec le nom `td3-db`, mais ce nom n'est pas résolu. Les trois conteneurs n'étant pas connectés à un même réseau Docker, ils ne peuvent pas communiquer correctement entre eux par leur nom.

La correction consiste à créer un réseau Docker dédié et à connecter les trois conteneurs à ce même réseau avec l'option `--network`.

****A2.**** Après plusieurs appels, les compteurs étaient à 4 dans Redis et PostgreSQL. J'ai ensuite supprimé les trois conteneurs puis je les ai recréés en conservant le même volume nommé `td3-db-data` pour PostgreSQL.

Après recréation, le résultat était :

```json
{"hitsInRedis":1,"visitsInPostgres":5,"servedBy":"d8168275c30a"}
```

Le compteur PostgreSQL a donc été conservé : il est passé de 4 à 5, car les données sont stockées dans le volume nommé `td3-db-data`, qui survit à la suppression du conteneur.

À l'inverse, le compteur Redis est revenu à 1, car aucun volume n'était associé au conteneur Redis. La suppression du conteneur a donc supprimé ses données.

Le champ `servedBy` a également changé, car un nouveau conteneur API a été créé.

****B1.****

J'ai créé un fichier `compose.yaml` contenant les trois services `api`, `db` et `cache`.

L'API est construite à partir du `Dockerfile` situé dans `app/` avec le target `prod`. PostgreSQL utilise l'image `postgres:18-alpine` et Redis utilise `redis:8-alpine`.

Les paramètres de connexion et le mot de passe PostgreSQL sont récupérés depuis le fichier `.env` grâce à l'interpolation des variables Docker Compose. Le mot de passe n'est donc pas écrit directement dans `compose.yaml`.

Le fichier `.env.example` est fourni avec des valeurs d'exemple et peut être versionné, contrairement au fichier `.env` qui contient les valeurs utilisées localement.

La commande `docker compose config` m'a permis de vérifier que les variables étaient correctement interpolées et que la configuration finale était valide.

****B2.****

Avec `docker compose up -d`, les services ont démarré correctement dans mon environnement. Les logs de l'API indiquaient :

```text
Connecting to Postgres at db:5432…
Connecting to Redis at redis://cache:6379…
Visites API listening on 3000
```

Cependant, `depends_on` ne garantit que l'ordre de démarrage des conteneurs et ne garantit pas que PostgreSQL ou Redis soient déjà prêts à accepter les connexions.

J'ai donc ajouté :

```yaml
restart: on-failure
```

au service `api`.

Cette configuration permet à l'API de redémarrer automatiquement si elle s'arrête à cause d'une erreur de connexion temporaire au démarrage.

Il s'agit toutefois d'une solution de contournement et non d'une véritable gestion de l'état de préparation des services. Une solution plus complète consiste à utiliser des `healthcheck` et à attendre que les services soient réellement prêts avant de démarrer l'application.

****B3.**** Avec `docker compose down` puis `docker compose up -d`, les données PostgreSQL ont été conservées grâce au volume nommé `db-data`. Avant l'ajout d'un volume Redis, son compteur était réinitialisé.

Après avoir ajouté un volume nommé `cache-data` monté sur le répertoire `/data`, les données Redis sont également conservées.

Après `docker compose down` puis `docker compose up -d`, j'ai obtenu :

```json
{"hitsInRedis":2,"visitsInPostgres":6,"servedBy":"5abb6805ac6a"}
```

Les deux compteurs ont donc survécu à la suppression et à la recréation des conteneurs.

En revanche, `docker compose down -v` supprime les volumes nommés du projet. Les données PostgreSQL et Redis sont alors perdues et les compteurs repartent de zéro.

****B4.**** Depuis le conteneur API, `getent hosts db` retournait :

```text
172.18.0.2        db  db
```

Après avoir recréé PostgreSQL avec `docker compose up -d --force-recreate db`, l'adresse obtenue était toujours `172.18.0.2`.

L'adresse IP n'a donc pas changé dans ce test. Cela ne signifie pas qu'elle est fixe : Docker peut simplement réattribuer la même adresse disponible au nouveau conteneur.

L'application utilise le nom de service `db` et non son adresse IP (`DB_HOST=db`). Docker Compose fournit automatiquement la résolution DNS entre les services. Il ne faut donc pas mettre l'IP d'un conteneur dans la configuration, car elle peut changer lors de la recréation du conteneur.

****C1.**** J'ai créé `compose.override.yaml` afin de passer l'API sur le target `dev`. L'API utilise alors `node --watch src/server.js`.

La configuration `develop.watch` contient une règle `sync` qui synchronise automatiquement le contenu de `app/src` vers `/app/src` dans le conteneur. Une modification du fichier `server.js` est donc prise en compte sans avoir à reconstruire manuellement l'image.

J'ai également configuré une règle `rebuild` sur `package.json` : si ce fichier est modifié, l'image est automatiquement reconstruite.

J'ai lancé :

```bash
docker compose up -d --build
docker compose watch
```

Après modification de la réponse de `/health`, le résultat a changé sans lancer de nouvelle commande de build, ce qui confirme que le mécanisme de synchronisation fonctionne.

La base PostgreSQL est également accessible depuis la machine grâce au port configurable avec la variable `DB_PORT_HOST` définie dans `.env`.

****C2.**** Pour repasser en production uniquement, j'ai utilisé :

```bash
docker compose -f compose.yaml up -d --build
```

Le fichier `compose.override.yaml` n'étant pas chargé, l'API utilise le target `prod`. Les logs ne contiennent plus `node --watch src/server.js` et indiquent :

```text
Connecting to Postgres at db:5432…
Connecting to Redis at redis://cache:6379…
Visites API listening on 3000
```

J'ai ensuite arrêté les conteneurs puis relancé la stack avec :

```bash
docker compose -f compose.yaml down
docker compose -f compose.yaml up -d
```

sans utiliser `--build`. L'application a tout de même démarré correctement, car une image correspondant déjà à la configuration de production était disponible localement.

Cependant, si le code source ou le `Dockerfile` a été modifié depuis la dernière construction de l'image, oublier `--build` peut faire utiliser une ancienne image. Les modifications ne seront alors pas présentes dans le conteneur.

Le paramètre `--build` permet donc de reconstruire l'image avant le démarrage des services afin de prendre en compte les dernières modifications.
