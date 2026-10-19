# DevOps — Rapport

## TP part 01 — Docker

Structure : un dossier par image dans `tp1/` (`database/`, `backend/`, `http-server/`) + `tp1/docker-compose.yml`.

### Database

`tp1/database/Dockerfile` :
```dockerfile
FROM postgres:17.2-alpine

COPY 01-CreateScheme.sql 02-InsertData.sql /docker-entrypoint-initdb.d/
```
Les scripts SQL copiés dans `/docker-entrypoint-initdb.d` sont exécutés par ordre alphabétique au premier démarrage (création des tables puis insertion des données).

Commandes :
```bash
docker network create app-network
docker build -t tp1-database tp1/database
docker run -d --name tp1-database --network app-network \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd \
  -v tp1-db-data:/var/lib/postgresql/data \
  tp1-database
docker run -d -p "8090:8080" --net=app-network --name=adminer adminer
```
Adminer est accessible sur http://localhost:8090 (serveur `tp1-database`).

**1-1 Pourquoi passer les variables d'environnement avec `-e` plutôt que dans le Dockerfile ?**
Le Dockerfile est versionné et les variables `ENV` sont stockées dans l'image (visibles avec `docker inspect` ou `docker history`) : les mots de passe seraient exposés à toute personne ayant accès au repo ou à l'image. Avec `-e`, les secrets restent hors de l'image et la même image peut être réutilisée dans plusieurs environnements (dev, prod) avec des valeurs différentes.

**1-2 Pourquoi attacher un volume au conteneur postgres ?**
Le système de fichiers d'un conteneur est éphémère : si le conteneur est supprimé, les données de `/var/lib/postgresql/data` sont perdues. Un volume stocke ces données en dehors du conteneur, sur l'hôte : elles survivent à la suppression/recréation du conteneur. Vérifié : ajout d'une ligne, `docker rm -f tp1-database`, recréation avec le même volume → la ligne est toujours présente.

**1-3 Essentiels du conteneur base de données**
- Image `postgres:17.2-alpine`, scripts d'init copiés dans `/docker-entrypoint-initdb.d`.
- Identifiants passés au `docker run` avec `-e`.
- Réseau `app-network` pour communiquer avec adminer et le backend par nom de conteneur.
- Volume `-v tp1-db-data:/var/lib/postgresql/data` pour la persistance.
- Le port 5432 n'est pas publié sur l'hôte.

### Backend API

**Basics** — `tp1/backend/hello/` :
```dockerfile
FROM eclipse-temurin:21-jre-alpine

COPY Main.class /

CMD ["java", "Main"]
```
```bash
javac --release 21 Main.java   # JDK local en 25 : on cible Java 21 pour le JRE de l'image
docker build -t tp1-hello tp1/backend/hello
docker run --rm --name tp1-hello tp1-hello   # affiche "Hello World!"
```

**Simple api** — `tp1/backend/simpleapi/` : projet Spring Initializr (Maven, Java 21, Spring Web) + `GreetingController`. Spring Initializr ne propose plus Spring Boot 3.4.5, le projet a donc été généré puis le `pom.xml` remis en 3.4.5 (`spring-boot-starter-web`).
```bash
docker build -t tp1-simpleapi tp1/backend/simpleapi
docker run -d --name tp1-simpleapi -p 8080:8080 tp1-simpleapi
curl "localhost:8080/?name=Nathan"   # {"id":2,"content":"Hello, Nathan!"}
```

**1-4 Pourquoi un build multistage ? Explication du Dockerfile**
Le multistage sépare la compilation de l'exécution : le JDK et Maven (lourds, nécessaires uniquement pour compiler) restent dans le premier stage, l'image finale ne contient qu'un JRE et le jar. L'image est plus légère, plus rapide à transférer et expose moins de surface d'attaque. On n'a pas non plus besoin de Java ou Maven sur la machine qui build.
```dockerfile
# Build stage
FROM eclipse-temurin:21-jdk-alpine AS myapp-build   # stage 1 : JDK 21, nommé myapp-build
ENV MYAPP_HOME=/opt/myapp                           # variable pour le dossier de l'app
WORKDIR $MYAPP_HOME                                 # dossier de travail
RUN apk add --no-cache maven                        # installe Maven
COPY pom.xml .                                      # copie la config Maven
COPY src ./src                                      # copie le code source
RUN mvn package -DskipTests                         # compile et produit le jar dans target/

# Run stage
FROM eclipse-temurin:21-jre-alpine                  # stage 2 : uniquement le JRE
ENV MYAPP_HOME=/opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar   # récupère le jar du stage 1
ENTRYPOINT ["java", "-jar", "myapp.jar"]            # lance l'application
```

**Backend API** — `tp1/backend/simple-api/` : `src` et `pom.xml` de [simple-api-student](https://github.com/takima-training/simple-api-student), même Dockerfile multistage. Dans `application.yml`, la datasource lit des variables d'environnement :
```yaml
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
```
```bash
docker build -t tp1-backend tp1/backend/simple-api
docker run -d --name tp1-backend --network app-network -p 8080:8080 \
  -e DB_URL=jdbc:postgresql://tp1-database:5432/db -e DB_USER=usr -e DB_PASSWORD=pwd \
  tp1-backend
curl localhost:8080/departments/IRC/students
# [{"id":1,"firstname":"Eli","lastname":"Copter","department":{"id":1,"name":"IRC"}}]
```

### Http server

`tp1/http-server/Dockerfile` :
```dockerfile
FROM httpd:2.4-alpine

COPY index.html /usr/local/apache2/htdocs/
COPY httpd.conf /usr/local/apache2/conf/httpd.conf
```
Configuration par défaut récupérée avec `docker cp tp1-httpd:/usr/local/apache2/conf/httpd.conf ./httpd.conf`, puis ajout en fin de fichier :
```apache
<VirtualHost *:80>
ProxyPreserveHost On
ProxyPass / http://backend:8080/
ProxyPassReverse / http://backend:8080/
</VirtualHost>
LoadModule proxy_module modules/mod_proxy.so
LoadModule proxy_http_module modules/mod_proxy_http.so
```
Commandes utilisées pour vérifier : `docker stats`, `docker inspect`, `docker logs`.

**1-5 Pourquoi un reverse proxy ?**
C'est le point d'entrée unique de l'application : seul le proxy est exposé, le backend et la base restent sur le réseau interne. Il permet de centraliser le SSL/HTTPS, de servir un front, de faire du load balancing entre plusieurs backends, et de changer l'architecture interne sans impacter les clients.

### Docker-compose

`tp1/docker-compose.yml` :
- `database` : build `./database`, identifiants via `.env`, volume nommé `db-data`, réseau `back-network` uniquement.
- `backend` : build `./backend/simple-api`, URL de la base `database:5432`, réseaux `back-network` (vers la base) et `front-network` (vers httpd), `depends_on: database`.
- `httpd` : build `./http-server`, seul service exposé (`80:80`), réseau `front-network`, `depends_on: backend`.
- `restart: unless-stopped` sur les 3 services ; les secrets sont dans `tp1/.env` (non commité).
- Segmentation : httpd ne peut pas joindre la base, et ni la base ni le backend ne sont exposés sur l'hôte.

**1-6 Pourquoi docker-compose est-il si important ?**
Il décrit toute l'application multi-conteneurs dans un seul fichier versionné (images, réseaux, volumes, variables, ordre de démarrage) et la pilote avec une seule commande au lieu d'une série de `docker run` manuels. Résultat reproductible sur n'importe quelle machine.

**1-7 Commandes docker-compose principales**
| Commande | Rôle |
|---|---|
| `docker compose up -d --build` | (re)build les images et démarre les services en arrière-plan |
| `docker compose ps` | état des services |
| `docker compose logs -f <service>` | logs d'un service |
| `docker compose stop` / `start` | arrête / redémarre sans supprimer |
| `docker compose down` | supprime conteneurs et réseaux (`-v` pour supprimer aussi les volumes) |
| `docker compose build` | build les images sans démarrer |
| `docker compose exec <service> <cmd>` | exécute une commande dans un service |

**1-8 Le fichier docker-compose** : voir la description ci-dessus et `tp1/docker-compose.yml`.

Vérification :
```bash
cd tp1 && docker compose up -d --build
curl localhost/departments/IRC/students   # passe par httpd -> backend -> database
curl localhost:8080                       # échoue : backend non exposé
```

### Publish

**1-9 Commandes de publication et images publiées**
```bash
docker login
docker tag tp1-database nathank04/tp1-database:1.0
docker tag tp1-backend  nathank04/tp1-backend:1.0
docker tag tp1-httpd    nathank04/tp1-httpd:1.0
docker push nathank04/tp1-database:1.0
docker push nathank04/tp1-backend:1.0
docker push nathank04/tp1-httpd:1.0
```
Images publiées sur Docker Hub (compte `nathank04`) :
| Image | Tag | Taille |
|---|---|---|
| [nathank04/tp1-database](https://hub.docker.com/r/nathank04/tp1-database) | 1.0 | ~111 Mo |
| [nathank04/tp1-backend](https://hub.docker.com/r/nathank04/tp1-backend) | 1.0 | ~123 Mo |
| [nathank04/tp1-httpd](https://hub.docker.com/r/nathank04/tp1-httpd) | 1.0 | ~23 Mo |

Le tag `1.0` donne une version explicite à l'image (au lieu de `latest`), ce qui permet de savoir exactement quelle version est déployée et de revenir en arrière.

**1-10 Pourquoi mettre nos images dans un registre en ligne ?**
Pour les partager avec l'équipe et les déployer sur d'autres machines (serveurs, CI/CD) sans rebuild : on récupère exactement la même image versionnée (tag) partout. Le registre sert aussi d'historique des versions et permet de revenir à une version précédente.
