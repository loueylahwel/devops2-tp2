# TP2 — Réponses

> Brouillon. Les questions qui demandent d'observer un test sur Docker (Q3, Q9, Q11, Q14) sont à
> compléter après exécution sur votre machine.

**Q1.** `backend/config/database.js` lit `DATABASE_CLIENT` (ici `postgres`), `DATABASE_HOST`,
`DATABASE_PORT`, `DATABASE_NAME`, `DATABASE_USERNAME`, `DATABASE_PASSWORD` (et `DATABASE_URL` en
option). `DATABASE_HOST` doit valoir `postgres`, le nom du service Compose : les conteneurs se
joignent par leur nom sur le réseau Docker. `localhost` désignerait le conteneur Strapi lui-même,
où PostgreSQL ne tourne pas.

**Q2.** `backend/.env.example` documente les variables attendues par Strapi, sans valeurs secrètes
réelles. Le vrai `.env` contient les secrets (clés, mots de passe) : il ne doit jamais être commité,
sinon ils sont publiés dans l'historique Git et il faut tous les régénérer.

**Q3.** `docker compose down` arrête et supprime les conteneurs et le réseau, mais garde les
volumes : les données PostgreSQL restent. `docker compose down -v` supprime aussi les volumes
(`pgdata`, etc.) : la base est perdue. *(À vérifier par un test.)*

**Q4.** Compose remplace les `${VAR}` de son fichier par les valeurs de `.env` **avant** de lancer le
conteneur. Le healthcheck `CMD-SHELL` est en revanche exécuté dans le conteneur. Le `$$` échappe le
`$` : Compose ne substitue pas `${POSTGRES_USER}`, et c'est le shell du conteneur qui le lit, avec la
variable d'environnement définie dans le conteneur.

**Q5.** `"Host": "postgres"` : pgAdmin se connecte à PostgreSQL par le nom du service, sur le réseau
Docker, comme Strapi. `:ro` (read-only) monte le fichier en lecture seule : pgAdmin peut le lire mais
ne peut pas le modifier. `${PGADMIN_PORT:-5050}` prend la valeur de `PGADMIN_PORT` si elle est
définie, sinon `5050` (valeur par défaut).

**Q6.** Docker met en cache chaque étape. Si `package*.json` ne change pas, l'étape `npm ci` est
reprise depuis le cache : on ne réinstalle pas les dépendances à chaque modification du code. Si l'on
modifie un fichier de `backend/config/`, seules les étapes après `COPY . .` sont rejouées : le
`npm run build` et la création de l'image.

**Q7.** `USER node` fait tourner le processus sans droits root : en cas de faille, l'attaquant a
des droits limités dans le conteneur. `--chown=node:node` donne à l'utilisateur `node` la propriété
des fichiers copiés, pour que l'application puisse les lire (et écrire, par exemple les uploads).

**Q8.** `condition: service_healthy` fait attendre Strapi que PostgreSQL passe son healthcheck
(`pg_isready`). Sans cela, Compose ne fait que démarrer le conteneur PostgreSQL : Strapi peut tenter
de se connecter alors que la base n'accepte pas encore de connexions, et échouer au démarrage.

**Q9.** Trois articles publiés donnent **six lignes** dans la table `articles`. Chaque article a une
ligne brouillon (`published_at` vide) et une ligne publiée (`published_at` renseigné). Les deux lignes
partagent le même `document_id` mais ont des `id` différents (1 et 4, 2 et 5, 3 et 6).

**Q10.** Dans le conteneur `frontend`, `localhost` désigne ce conteneur, où il n'y a pas de Strapi.
Le front doit appeler `http://strapi:1337` (nom du service, sur le réseau `frontend`).

**Q11.** Le Dockerfile du front a trois étapes (`deps`, `builder`, `runner`). L'image finale ne
contient que `public/`, le `.next/standalone` (serveur Node minimal avec les dépendances de
production) et `.next/static`. Sans `output: "standalone"`, le dossier `.next/standalone` n'existe pas
et le `COPY` échoue au build. La taille observée de l'image finale `tp2/frontend` est de **334 Mo** (`docker images tp2/frontend`).

**Q12.** `strapi` est dans les deux réseaux : il doit parler à PostgreSQL (`backend`) et être
joignable par le front (`frontend`). Ne pas publier de port pour `postgres` réduit la surface
d'attaque : la base n'est accessible qu'aux conteneurs des réseaux auxquels elle appartient, et non
depuis la machine hôte ou d'autres réseaux.

**Q13.** Par exemple : (1) ne pas utiliser de secrets en clair dans `.env`, mais un gestionnaire de
secrets (Docker secrets, coffre externe) ; (2) exécuter les images avec un système de fichiers en
lecture seule et sans privilèges supplémentaires (`read_only`, `cap_drop: [ALL]`) ; (3) épingler les
versions d'images par digest et scanner les images (vulnérabilités) ; (4) TLS devant le front et
Strapi via un reverse proxy.

**Q14.** `unless-stopped` redémarre le conteneur après un crash ou au redémarrage de Docker, sauf s'il
a été arrêté volontairement. `docker compose stop` ou `kill` est un arrêt volontaire : Docker ne le
relance pas. Il le relance automatiquement seulement après un arrêt non voulu (crash, erreur, reboot
de la machine si Docker redémarre).
