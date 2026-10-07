## Partie A — Premiers conteneurs

**A1.** Lancez `docker run hello-world`. Relancez-le une seconde fois.
> Qu'est-ce qui diffère entre les deux exécutions, et pourquoi ?

```txt
La première fois, il n'arrive pas à trouver l'image "hello-world" localement et donc la récupère depuis la bibliothèque puis lance le conteneur qui affiche un message dans le terminal. La seconde fois, puisque l'image est déjà téléchargé, il lance directement le conteneur et affiche le message dans le terminal.
```

**A2.** Lancez un nginx en arrière-plan, nommé `web1`, accessible sur http://localhost:8080 (image `nginx:1.29-alpine`).
Lancez-en un second, `web2`, accessible sur http://localhost:8081.
> Les deux nginx écoutent sur le port 80 **dans** leur conteneur. Pourquoi n'y a-t-il pas de conflit ?
> Que se passe-t-il si vous essayez de publier `web2` aussi sur 8080 ?

```bash
docker run -d --name web1 -p 8080:80 nginx:1.29-alpine
docker run -d --name web2 -p 8081:80 nginx:1.29-alpine
docker stop web2
docker rm web2
docker run -d --name web2 -p 8080:80 nginx:1.29-alpine
```

```txt
Les deux nginx ne sont pas sur le même port hôte donc ils peuvent tout les deux utilisé le port 80.
Lorsque on essaye de publier web2 sur 8080, Docker refuse car le port est déjà utilisé.
```

**A3.** Rafraîchissez plusieurs fois http://localhost:8080, puis affichez les logs de `web1`. Suivez-les en continu
pendant que vous rafraîchissez.
> Quelle commande ? D'où viennent ces lignes ?

```bash
docker logs -f web1
```

```txt
Le navigateur envoie des requêtes comme GET / HTTP/1.1 et reçoie le code 200 (OK) la première fois, puis pour GET /favicon.ico HTTP/1.1 reçoit un 404 (Not Found) car le fichier est absent, puis de nouveau pour GET / HTTP/1.1 reçoit des codes 304 (Not Modified).
```

**A4.** Entrez dans `web1` et remplacez le contenu de `/usr/share/nginx/html/index.html` par votre prénom.
Vérifiez dans le navigateur. Supprimez `web1`, puis relancez-le avec **exactement** la même commande qu'en A2.
> Où est passée votre modification ? Qu'en concluez-vous sur l'usage de `docker exec` pour modifier une application ?

```txt
On entre dans le conteneur avec :
docker exec -it web1 sh

puis on remplace le contenu du fichier :
echo "Quentin" > /usr/share/nginx/html/index.html

On vérifie dans le navigateur : la page affiche bien le prénom. Cependant, quand on supprime `web1` puis on le relance avec exactement la même commande qu'en A2, la modification a disparu.

La modification n'est pas persistée dans l'image, car le conteneur est juste une instance d'exécution de l'image. `docker exec` permet d'entrer dans un conteneur en cours d'exécution pour faire un diagnostic ou une petite modification temporaire, mais cela ne modifie pas définitivement l'application ni l'image. Dès que le conteneur est supprimé, son système de fichiers est perdu.

On conclut que `docker exec` n'est pas un bon moyen pour modifier durablement une application ; il faut plutôt modifier le contenu de l'image via un Dockerfile, ou recréer un conteneur à partir d'une image modifiée.
```

## Partie B — Variables d'environnement et mode interactif

**B1.** Lancez un conteneur `alpine` jetable en lui passant une variable d'environnement `PRENOM` (option `-e`),
et faites-lui afficher cette variable avec la commande `printenv PRENOM`.
Recommencez **sans** l'option `-e`.
> Que s'affiche-t-il dans chaque cas ? La variable `PRENOM` existe-t-elle sur votre machine (`echo $PRENOM`) ?
> Où « vit » une variable passée avec `-e` ?

```bash
docker run --rm -e PRENOM=Quentin alpine printenv PRENOM
docker run --rm alpine printenv PRENOM
echo $PRENOM
```

```txt
Avec l'option -e, la variable PRENOM est créée dans le conteneur et sa valeur est affichée dans le terminal :

Quentin

Sans l'option -e, la variable PRENOM n'existe pas dans le conteneur donc rien ne s'affiche.

La commande echo $PRENOM sur ma machine n'affiche également rien, car la variable passée avec -e n'est pas créée sur ma machine.

La variable passée avec -e vit uniquement dans l'environnement du conteneur. Elle est disponible pendant l'exécution du conteneur mais n'est pas une variable d'environnement de la machine hôte.

L'option --rm supprime ensuite le conteneur une fois la commande terminée.
```

**B2.** Ouvrez un shell interactif dans un conteneur `alpine` jetable (`--rm`). Installez `curl` (`apk add curl`)
et vérifiez qu'il marche. Quittez, relancez la même commande.
> `curl` est-il encore là ? Pourquoi ? Dans quoi l'installation avait-elle été faite : l'image ou le conteneur ?
> (Comment l'installer durablement : réponse cet après-midi, avec le Dockerfile.)

```bash
docker run --rm -it alpine sh
apk add curl
curl --version
exit
docker run --rm -it alpine sh
curl --version
```

```txt
Après avoir installé curl avec apk add curl, la commande fonctionne bien dans le premier conteneur.

Lorsque je quitte le conteneur puis que je relance la même commande avec --rm, curl n'est plus présent.

L'installation avait été faite dans le conteneur et non dans l'image alpine. Le conteneur avait donc été modifié pendant son exécution, mais l'image d'origine n'a pas été modifiée.

Comme le conteneur avait été lancé avec --rm, il est supprimé lorsque je quitte le shell. Toutes les modifications faites à l'intérieur sont donc perdues.

Pour installer curl durablement, il faudra créer une nouvelle image avec un Dockerfile qui installe curl.
```

## Partie C — Images et couches

**C1.** Téléchargez `node:24`, `node:24-slim` et `node:24-alpine`, puis comparez leurs tailles (`docker image ls node`).
Pour comprendre l'écart, regardez ce que contient chacune (remplacez `node:24` par les deux autres) :
```bash
docker run --rm node:24 sh -c 'ls /usr/bin | wc -l'   # nombre de commandes installées
docker run --rm node:24 which gcc git curl            # ces outils sont-ils présents ?
```
> Tableau : taille, nombre de commandes, outils présents. Les trois font tourner le même Node 24 :
> qu'est-ce que la plus grosse contient en plus ? Ces outils sont-ils utiles pour **faire tourner** une API ?

```bash
docker pull node:24
docker pull node:24-slim
docker pull node:24-alpine
docker image ls node

docker run --rm node:24 sh -c 'ls /usr/bin | wc -l'
docker run --rm node:24-slim sh -c 'ls /usr/bin | wc -l'
docker run --rm node:24-alpine sh -c 'ls /usr/bin | wc -l'
docker run --rm node:24 which gcc git curl
docker run --rm node:24-slim which gcc git curl
docker run --rm node:24-alpine which gcc git curl
```

```txt
Les trois images utilisent la même version principale de Node.js 24, mais elles n'ont pas la même taille.

| Image          | Taille | Nombre de commandes | Outils présents |
|----------------|--------|---------------------|-----------------|
| node:24        | plus grande | plus important | gcc, git, curl... |
| node:24-slim   | moyenne | moins important | moins d'outils |
| node:24-alpine | plus petite | encore moins important | très peu d'outils |

L'image node:24 est la plus grosse car elle contient beaucoup plus de logiciels et d'outils système, notamment des outils de compilation et différents utilitaires.

Ces outils peuvent être utiles pour développer ou compiler une application, mais ils ne sont généralement pas nécessaires pour simplement faire tourner une API Node.js déjà construite.

C'est pour cela que les images slim ou alpine peuvent être intéressantes en production : elles contiennent moins de choses inutiles et sont donc plus légères.
```

**C2.** Affichez l'historique des couches de `node:24-alpine`.
> Combien de couches ? Quelle instruction a produit la plus lourde ?

```bash
docker history node:24-alpine
```

```txt
La commande docker history permet d'afficher les différentes couches qui composent l'image.

On peut voir plusieurs couches correspondant aux différentes instructions utilisées pour construire l'image.

La couche la plus lourde correspond à l'ajout et à l'installation des fichiers principaux de l'image, car c'est celle qui contient la majorité du système nécessaire pour faire fonctionner Node.js.

Les couches permettent de construire l'image progressivement et Docker peut réutiliser les couches déjà téléchargées lorsqu'elles sont identiques.
```

**C3.** Une image contient aussi des **métadonnées**. Affichez-les :
```bash
docker image inspect nginx:1.29-alpine
```
Dans ce (long) résultat, retrouvez les champs `Cmd` et `ExposedPorts` (recherche dans le terminal, ou
`docker image inspect nginx:1.29-alpine | grep -A4 Cmd`).
> Quelle commande est lancée au démarrage d'un conteneur nginx ? Quel port est indiqué ?
> Est-ce cohérent avec le port que vous avez utilisé en A2 ?

```txt
Dans les métadonnées de l'image, on retrouve la commande utilisée pour démarrer nginx ainsi que le port exposé.

La commande de démarrage correspond à nginx avec le paramètre permettant de le laisser fonctionner au premier plan :

nginx -g daemon off;

Le port indiqué dans ExposedPorts est le port 80.

C'est cohérent avec la partie A2 car nginx écoute bien sur le port 80 à l'intérieur du conteneur.

En A2, le port 8080 de ma machine était redirigé vers le port 80 du conteneur avec :

-p 8080:80

Le premier port correspond donc à la machine et le deuxième au conteneur.
```

## Partie D — Énigmes

Pour chaque énigme : **ce que vous observez**, **l'explication**, **la correction**.

**D1.** `docker run -d alpine` rend la main… mais `docker ps` n'affiche rien.
(Indices : `docker ps -a`, et le `Cmd` de l'image `alpine`, comme en C3.)

```bash
docker run -d alpine
docker ps
docker ps -a
```

```txt
Ce que j'observe :

La commande docker run -d alpine rend immédiatement la main et le conteneur n'apparaît pas avec docker ps.

Avec docker ps -a, je peux voir que le conteneur existe mais qu'il est arrêté avec le statut Exited.

L'explication :

L'image alpine lance par défaut une commande qui se termine immédiatement lorsqu'elle n'a rien à faire.

Comme la commande principale du conteneur se termine, le conteneur s'arrête également.

Le -d signifie seulement que le conteneur est lancé en arrière-plan. Cela ne signifie pas qu'il va rester en fonctionnement.

La correction :

Il faut lancer une commande qui reste active, par exemple :

docker run -d alpine sleep 1000

Le conteneur reste alors actif pendant l'exécution de sleep.

On peut également utiliser un shell interactif avec :

docker run -it alpine sh
```

**D2.** `docker run -d -p 9082:8080 nginx:1.29-alpine` : le conteneur tourne, mais http://localhost:9082 ne répond pas.

```bash
docker run -d -p 9082:8080 nginx:1.29-alpine
```

```txt
Ce que j'observe :

Le conteneur nginx fonctionne bien et apparaît dans docker ps, mais lorsque j'accède à http://localhost:9082, la page ne répond pas.

L'explication :

Le problème vient du mapping des ports.

La commande :

-p 9082:8080

signifie que le port 9082 de ma machine est redirigé vers le port 8080 du conteneur.

Cependant, nginx écoute sur le port 80 dans le conteneur et non sur le port 8080.

Docker redirige donc les requêtes vers le port 8080, mais aucun service nginx n'écoute sur ce port.

La correction :

Il faut rediriger le port 9082 de la machine vers le port 80 du conteneur :

docker run -d -p 9082:80 nginx:1.29-alpine

On peut alors accéder à nginx avec :

http://localhost:9082
```

**D3.** Chronométrez l'arrêt de ce conteneur, puis regardez son code de sortie :
```bash
docker run -d --name dormeur alpine sleep 1000
time docker stop dormeur
docker ps -a --filter name=dormeur          # colonne STATUS : « Exited (…) »
```
> Combien de temps a pris l'arrêt (selon les machines, de quelques secondes à 10 s) ? Quel code de sortie ? À l'aide de la slide « Cycle de vie », expliquez
> ce que Docker a fait pendant ces secondes. (Pourquoi `sleep` n'a pas réagi : on y reviendra plus tard.)
>
> *Bonus :* recommencez avec `docker run -d --init --name dormeur2 alpine sleep 1000`. Qu'est-ce qui change ?

```bash
docker run -d --name dormeur alpine sleep 1000

time docker stop dormeur

docker ps -a --filter name=dormeur
```

```txt
Ce que j'observe :

Le conteneur démarre avec la commande sleep 1000.

Lorsque je fais docker stop dormeur, Docker met plusieurs secondes avant de terminer l'arrêt du conteneur.

Le conteneur apparaît ensuite avec le statut :

Exited (137)

L'explication :

Lorsque docker stop est utilisé, Docker commence par envoyer un signal SIGTERM au processus principal du conteneur afin de lui demander de s'arrêter proprement.

Docker attend ensuite quelques secondes pour lui laisser le temps de terminer.

Comme le processus sleep ne réagit pas au signal SIGTERM comme attendu, Docker finit par envoyer un signal SIGKILL pour forcer son arrêt.

Le processus est donc tué et le conteneur se termine avec le code de sortie 137.

Le code 137 correspond à un processus terminé par SIGKILL (9), car 128 + 9 = 137.

La durée observée correspond donc au temps pendant lequel Docker attend avant de forcer l'arrêt du conteneur.
```

**D4.** Lancez :
```bash
docker run --name gourmand --memory 50m node:24-alpine \
  node -e "const a=[]; while(true) a.push(new Array(1e6).fill(1))"
```
> Que se passe-t-il ? Quel code de sortie ? Trouvez dans `docker inspect gourmand` le champ qui le confirme.
> Quel mécanisme du noyau vu en cours est à l'œuvre ?

```bash
docker run --name gourmand --memory 50m node:24-alpine \
node -e "const a=[]; while(true) a.push(new Array(1e6).fill(1))"
```

```txt
Ce que j'observe :

Le programme Node.js consomme de plus en plus de mémoire jusqu'à atteindre la limite de 50 Mo définie avec --memory.

Le processus est ensuite arrêté automatiquement par Docker.

Le conteneur se termine avec un code de sortie 137.

On peut vérifier cette information avec :

docker inspect gourmand

Dans le résultat, on retrouve notamment :

"OOMKilled": true

Cela confirme que le conteneur a été arrêté car il a dépassé la limite de mémoire qui lui était attribuée.

L'explication :

Docker utilise les mécanismes du noyau Linux permettant de limiter et contrôler les ressources utilisées par les processus, notamment les cgroups.

Avec :

--memory 50m

le conteneur ne peut pas utiliser plus d'environ 50 Mo de mémoire.

Lorsque le programme Node.js dépasse cette limite, le mécanisme OOM (Out Of Memory) intervient et le processus est tué.

Le champ OOMKilled à true dans docker inspect confirme que le conteneur a été arrêté à cause d'un dépassement de la limite mémoire.
```

## Partie E — Ménage

**E1.** Combien d'espace Docker occupe-t-il sur votre machine ? Supprimez tous les conteneurs arrêtés de ce TD.
> Commandes utilisées, espace avant / après.

```bash
docker system
docker ps -a
docker container prune
```

```txt
La commande docker system df permet de voir l'espace utilisé par Docker sur ma machine.

Elle affiche notamment l'espace utilisé par les images, les conteneurs, les volumes et le cache de build.

Avant le nettoyage, j'ai relevé l'espace utilisé par Docker avec :

docker system df

J'ai ensuite supprimé les conteneurs arrêtés avec :

docker container prune

Après le nettoyage, j'ai relancé :

docker system df

Cela permet de comparer l'espace utilisé avant et après la suppression des conteneurs arrêtés.

Les images ne sont pas supprimées par docker container prune, seules les données des conteneurs arrêtés sont supprimées.
```

---

### Barème (/10)

| Partie | Points |
|---|---|
| A | 3 |
| B | 1,5 |
| C | 1,5 |
| D | 3,5 |
| E | 0,5 |
