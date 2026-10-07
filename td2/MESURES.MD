## Étape 1 — La version naïve

> **Q1.** Le fichier `.env` du dossier contient une clé d'API. Est-il dans votre image ? Pourquoi est-ce grave, même si l'image n'est « que » sur votre machine pour l'instant ?

Oui, le fichier `.env` est présent dans l'image car le Dockerfile utilise `COPY . .`, ce qui copie tout le dossier dans l'image.

C'est grave car le fichier `.env` contient une clé d'API. Une clé secrète ne doit pas être intégrée dans une image Docker : toute personne ayant accès à l'image peut potentiellement récupérer cette clé. Elle pourrait également se retrouver dans un registre Docker si l'image était publiée.

## Étape 2 — Le cache

> **Q2.** Pourquoi le rebuild est-il plus rapide ? Que se passe-t-il si vous modifiez `package.json` / `pom.xml` ?

Le rebuild est plus rapide car le Dockerfile v2 copie d'abord les fichiers `package.json` et `package-lock.json`, puis installe les dépendances avec `npm ci`, avant de copier le reste du code source.

Lorsque seul le code source est modifié, Docker peut réutiliser la couche contenant `npm ci` grâce au cache. Il n'est donc pas nécessaire de réinstaller toutes les dépendances.

Si `package.json` ou `package-lock.json` est modifié, la couche `COPY package*.json ./` est modifiée. Docker doit alors réexécuter `npm ci`, car les dépendances peuvent avoir changé.

On observe cette différence dans les mesures : le rebuild passe de **9,235 s en v1 à 4,194 s en v2**.

## Étape 3 — `.dockerignore`

> **Q3.** Qu'avez-vous exclu, et pourquoi chaque ligne ? Regardez la ligne `transferring context` du build avant/après : qu'est-ce qui a changé ?

J'ai ajouté un fichier `.dockerignore` contenant `.env`, `node_modules`, `dist`, `.git` et `.gitignore`.

* `.env` est exclu car il contient des informations sensibles, notamment une clé d'API.
* `node_modules` est exclu car les dépendances sont installées directement dans l'image avec `npm ci`.
* `dist` est exclu car il s'agit d'un artefact généré par la compilation et il sera recréé avec `npm run build`.
* `.git` est exclu car l'historique Git n'est pas nécessaire au fonctionnement de l'application.
* `.gitignore` n'est pas nécessaire dans l'image.

Le contexte de build reste très petit. Surtout, les fichiers et dossiers inutiles ou potentiellement sensibles ne sont plus envoyés à Docker. Le fichier `.env` n'est également plus présent dans l'image, comme le confirme la commande :

`docker run --rm td2:v3 ls -la`

Le `.dockerignore` permet donc également d'éviter d'envoyer des fichiers sensibles ou inutiles au démon Docker.

## Étape 4 — Multi-stage

> **Q4.** Qu'est-ce qui est dans l'image de build et **n'est plus** dans l'image finale ? Citez-en au moins trois.

La v4 utilise un build multi-stage avec deux images. La première étape utilise `node:24` pour installer toutes les dépendances et compiler l'application TypeScript. Après la compilation, les dépendances de développement sont supprimées avec `npm prune --omit=dev`.

La deuxième étape utilise `node:24-alpine`, qui est une image plus légère. Elle ne récupère que les fichiers nécessaires à l'exécution : `dist`, les dépendances de production dans `node_modules` et les fichiers `package`.

Plusieurs éléments présents dans l'image de build ne sont donc plus présents dans l'image finale :

* le code source TypeScript situé dans `src/` ;
* les dépendances de développement ;
* les outils nécessaires à la compilation TypeScript ;
* les autres fichiers du projet qui ne sont pas nécessaires à l'exécution.

Cela permet de réduire fortement la taille de l'image. Elle passe ainsi de **1,7 GB pour les versions précédentes à 247 MB**.

Enfin, `USER node` permet d'exécuter l'application avec un utilisateur non privilégié au lieu de `root`, ce qui améliore la sécurité.

## Étape 5 — Une image, plusieurs configurations

> **Q5.** Pourquoi est-il important de ne **pas** reconstruire l'image pour changer le message ?

Les deux conteneurs utilisent la même image Docker `td2:v4`, mais ils ont des variables d'environnement différentes. Le premier conteneur utilise le message `Bonjour depuis le conteneur A` et la version `1.0.0`, tandis que le second utilise `Bonjour depuis le conteneur B` et la version `2.0.0`.

Les deux conteneurs répondent correctement aux routes `/` et `/health`. Cela montre que la configuration de l'application peut être fournie au démarrage du conteneur avec `-e`, sans avoir besoin de reconstruire l'image.

Il est donc préférable de garder l'image indépendante de la configuration et de fournir les valeurs comme `MESSAGE`, `APP_VERSION` ou d'autres paramètres au moment de l'exécution. Cela permet de réutiliser la même image dans différents environnements et d'éviter de reconstruire l'image pour chaque configuration.

# MESURES

| Version | Taille image | Build à froid | Rebuild après modification | `.env` dans l'image ? | Utilisateur         |
| ------- | -----------: | ------------: | -------------------------: | --------------------- | ------------------- |
| v1      |       1,7 GB |       8,966 s |                    9,235 s | Oui                   | root                |
| v2      |       1,7 GB |       8,545 s |                    4,194 s | Oui                   | root                |
| v3      |       1,7 GB |       9,343 s |                          — | Non                   | root                |
| v4      |       247 MB |       6,469 s |                    7,873 s | Non                   | node (UID 1000)     |
