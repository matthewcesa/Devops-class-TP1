# Devops-class-TP1
Ce TP déploie une application en trois couches avec Docker : une API Spring Boot, une base PostgreSQL initialisée par scripts SQL et un serveur Apache en reverse proxy, orchestrés avec Docker Compose.

Nom de l'image de la base de données : eucko/dbtp1 
Nom du conteneur dbtp1
 
### ** question 1-1 Pour quelle raison est-il préférable d'exécuter le conteneur avec un indicateur -e pour fournir les variables d'environnement plutôt que de les placer directement dans le Dockerfile ?**
Il est préférable d'exécuter le conteneur avec un indicateur -e car ainsi on peut réutiliser la même image dans différents environnements (dev, prod...) sans avoir à la reconstruire, et sans exposer de données sensibles (comme les mots de passe) directement dans le Dockerfile.

### ** question 1-2 Pourquoi avons-nous besoin d'un volume à attacher à notre conteneur postgres ?**
On a besoin de volume car ça nous permet de conserver les données même si le conteneur est supprimé ou recréé, car sans lui elles seraient perdues à chaque fois.

### ** question 1-3 Documentez les éléments essentiels de votre conteneur de base de données : commandes et Dockerfile. **
Arborescence :
```text
Devops-class-TP1/
├── docker-compose.yml
├── README.md
├── database/
│   ├── Dockerfile
│   └── init-db/
│       ├── CreateScheme.sql
│       └── InsertData.sql
├── network/
│   ├── Dockerfile
│   ├── httpd.conf
│   └── index.html
└── web/..
```

Les fichiers .sql sont des fichiers d'initialisation. 

Mon Dockerfile pour la base de données : 
```
FROM postgres:17.2-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=usr \
    POSTGRES_PASSWORD=pwd

COPY *.sql /docker-entrypoint-initdb.d/ 
```

Ensuite on a créé le réseau : docker network create app-network
On a construit l'image Docker : docker build -t dbtp1 ./database
Ensuite, on créé et lance le conteneur : 
```
docker run -d \\
--name dbtp1 \\
--network app-network \\
-e POSTGRES_DB=db \
-e POSTGRES_USER=usr \
-e POSTGRES_PASSWORD=pwd \
-v my-db-data:/var/lib/postgresql/data \
dbtp1
```
On vérifie ensuite les journaux et les tables créées :
```sh
docker logs dbtp1
Et on vérifie derrière si les données ont bien été transmises en créant un nouveau conteneur avec Adminer : 
```
```
docker run -d \
--name adminer \
--net=app-network \
-p 8090:8080 \
adminer
```
Je regarde combien de conteneur tourne avec docker ps (en l'occurence 2 ici).
On vérifie donc sur le localhost:8090 en se connectant à la base de données et j'ai effectivement des tables quand je rentre les informations necessaires.

## API BACKEND

### ** question 1-4 Pourquoi avons-nous besoin d'une construction en plusieurs étapes ? Et expliquez chaque étape de ce dockerfile. ** 
Le build multi-stage est important pour réduire la taille de l'image finale en ne gardant que le JRE léger et le fichier JAR, ce qui améliore la sécurité en retirant le code source et le JDK, tout en garantissant un build reproductible.
Dans la première phase de build, FROM eclipse-temurin:21-jdk-alpine AS myapp-build lance une image avec JDK 21, puis ENV MYAPP_HOME=/opt/myapp et WORKDIR $MYAPP_HOME créent et définissent le dossier de travail. 
L'instruction RUN apk add --no-cache maven installe Maven, COPY simpleapi/pom.xml . et COPY simpleapi/src ./src copient la configuration et les sources, puis RUN mvn clean package -DskipTests compile le projet en fichier JAR.   
Dans la seconde phase d'exécution, FROM eclipse-temurin:21-jre-alpine part d'une image vierge avec uniquement le JRE 21, ENV et WORKDIR réinstallent le dossier de travail, COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar récupère uniquement le JAR généré, et ENTRYPOINT ["java", "-jar", "myapp.jar"] lance l'application.


### **1-5 Why do we need a reverse proxy?**
Un reverse proxy sert d'intermédiaire sécurisé entre les utilisateurs et votre serveur d'application. Il sert à masquer la strutcure interne du réseau pour protéger le backend et permet de rediriger le trafic ou de repartir la charge entre plusieurs serveur tout en restant le seul point d'entree visible.

### **1-6 Pourquoi docker-compose est-il si important ?**
Permet d'automatiser et de regrouper la gestion de plusieurs conteneurs dans un seul fichier YAML, en effet au lieu de taper plein de commandes docker run, une seule commande (docker compose up) suffit pour tout lancer, connecter et configurer en même temps. C'est une automatisation. 

### **1-7 Documentation des commandes les plus importantes de docker-compose.**
Selon moi, les commandes les plus importantes sont : 
- docker compose up -d (demarrer en arrière-plan)
- docker compose down (arrêter et nettoyer tout) 
- docker compose ps (liste les services actifs)
- docker compose build (reconstruit les images).
```sh
docker compose config                 # valider le fichier compose
docker compose build                  # construire les images
docker compose up -d                  # demarrer les services en arrière-plan
docker compose up -d --build          # reconstruire puis démarrer
docker compose ps                     # voir l'état des services
docker compose logs -f                # suivre tous les journaux
docker compose logs -f backend        # suivre les journaux du backend
docker compose exec database psql -U usr -d db -c '\\dt'
docker compose down                   # arreter et supprimer les conteneurs/réseaux
```

### **1-8 Documentez votre fichier docker-compose.**
Le fichier `docker-compose.yml` orchestre trois services : `backend`, construit depuis `./web`, `database`, construit depuis `./database`, et `httpd`, construit depuis `./network`. 

Ils communiquent sur le réseau Docker `app-network` de type bridge. 
Le backend dépend de la base de données; Apache dépend du backend et publie le port 80 du conteneur sur le port 8080 de la machine, accessible à `http://localhost:8080`.

Les identifiants PostgreSQL (`db`, `usr`, `pwd`) sont définis dans le Dockerfile de la base pour correspondre à la configuration de connexion de l'API.
 Le volume `db-data` est déclaré dans Compose, mais le gabarit demandé ne prévoit pas de montage `volumes` dans le service `database`; il n'est donc pas utilisé pour conserver les données.

### **1-9 Documentez vos commandes de publication et vos images publiées sur Docker Hub**
```sh
docker login
docker build -t eucko/database ./database
docker build -t eucko/backend ./web
docker build -t eucko/httpd: ./network
docker push eucko/database
docker push eucko/backend
docker push eucko/httpd
```

Images publiées : eucko/database (PostgreSQL + scripts), eucko/backen: (API Spring Boot) et eucko/httpd:1.0 (Reverse Proxy Apache).

### **1-10 Pourquoi déposons-nous nos images dans un dépôt en ligne ?**
Déposer les images sur un registre en ligne permet de les partager entre collaborateurs, d'assurer le versioning des livrables (via les tags), de faciliter les déploiements automatisés (CI/CD) et de séparer l'étape de construction (Build) de celle d'exécution (Run).