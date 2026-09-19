# Devops-class-TP1
DevOps course – TP1: Reviewing Docker

## Base de données : 
Nom de l'image de la base de données : eucko/dbtp1 
Nom du conteneur dbtp1
 
### ** question 1-1 Pour quelle raison est-il préférable d'exécuter le conteneur avec un indicateur -e pour fournir les variables d'environnement plutôt que de les placer directement dans le Dockerfile ?**
Il est préférable d'exécuter le conteneur avec un indicateur -e car ainsi on peut réutiliser la même image dans différents environnements (dev, prod...) sans avoir à la reconstruire, et sans exposer de données sensibles (comme les mots de passe) directement dans le Dockerfile.

### ** question 1-2 Pourquoi avons-nous besoin d'un volume à attacher à notre conteneur postgres ?**
On a besoin de volume car ça nous permet de conserver les données même si le conteneur est supprimé ou recréé, car sans lui elles seraient perdues à chaque fois.

### ** question 1-3 Documentez les éléments essentiels de votre conteneur de base de données : commandes et Dockerfile. **
Arborescence 
Devops-class-TP1 
        database
                CreateScheme.sql
                InsertData.sql
                DockerFile
        network
        web
        README.md

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
Docker run -d \
--name dbtp1 \  
--net=app-network \
-e POSTGRES_DB=db \
-e POSTGRES_USER=usr \
-e POSTGRES_PASSWORD=pwd \
-v my-db-data:/var/lib/postgresql/data \
dbtp1
```
Et on vérifie derrière si les données ont bien été transmises en créant un nouveau conteneur avec Adminer : 
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


