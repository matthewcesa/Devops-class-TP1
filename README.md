# Devops-class-TP1
DevOps course – TP1: Reviewing Docker

## Base de données : 
Nom de l'image de la base de données : eucko/dbtp1 
Nom du conteneur dbtp1
 

question 1-1 Pour quelle raison est-il préférable d'exécuter le conteneur avec un indicateur -e pour fournir les variables d'environnement plutôt que de les placer directement dans le Dockerfile ?

Il est préférable d'exécuter le conteneur avec un indicateur -e car ainsi on peut réutiliser la même image dans différents environnements (dev, prod...) sans avoir à la reconstruire, et sans exposer de données sensibles (comme les mots de passe) directement dans le Dockerfile.