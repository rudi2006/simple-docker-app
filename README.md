Projet DevOps — Docker, CI/CD et Ansible

Projet réalisé dans le cadre des travaux pratiques DevOps. L'objectif est de conteneuriser une application en trois tiers, de publier ses images, puis d'automatiser son déploiement sur un serveur distant.

Sommaire

1. Objectifs et architecture

2. TP1 — Docker et application trois tiers

3. TP2 — Images et intégration continue avec GitHub Actions

4. TP3 — Déploiement distant avec Ansible

5. Configuration et structure du projet

6. Vérification de l'application déployée

7. Questions du TP Docker

8. Sécurité et améliorations possibles

9. Dépannage

1. Objectifs et architecture

Le projet met en place une application web composée de quatre conteneurs principaux :

PostgreSQL stocke les départements et les étudiants.

Backend Spring Boot expose une API REST en Java 21.

Frontend Vue.js servi par Nginx affiche les données provenant de l'API.

Apache HTTP Server (httpd) sert de reverse proxy et constitue le point d'entrée HTTP public.

Le déploiement est automatisé avec Ansible. GitHub Actions exécute les tests et, selon la configuration finale du workflow, construit et publie les images Docker sur Docker Hub avant de lancer le playbook de déploiement.

Navigateur
    |
    | HTTP :80
    v
Apache HTTPD — reverse proxy
    |                             \
    | /departments, /students      \ /
    v                               v
Backend Spring Boot :8080        Frontend Nginx :80
    |
    | réseau Docker privé
    v
PostgreSQL :5432

Le port public utilisé est le port 80 du serveur. PostgreSQL et le backend ne doivent pas exposer directement leurs ports à Internet : les communications entre conteneurs passent par les réseaux Docker.

Serveur de démonstration : http://kadjio.mballa.takima.school/
Dépôt GitHub : https://github.com/rudi2006/simple-docker-app

2. TP1 — Docker et application trois tiers

2.1 Base de données PostgreSQL

L'image de base utilisée est postgres:17.2-alpine. L'image personnalisée copie les scripts SQL dans le répertoire d'initialisation officiel de PostgreSQL.

Exemple de database/Dockerfile :

FROM postgres:17.2-alpine
COPY init/ /docker-entrypoint-initdb.d/

Les scripts d'initialisation créent deux tables : departments et students, puis insèrent les données de démonstration.

CREATE TABLE public.departments (
    id   SERIAL PRIMARY KEY,
    name VARCHAR(20) NOT NULL
);

CREATE TABLE public.students (
    id            SERIAL PRIMARY KEY,
    department_id INT NOT NULL REFERENCES departments(id),
    first_name    VARCHAR(20) NOT NULL,
    last_name     VARCHAR(20) NOT NULL
);

Données de démonstration :

Département

Étudiants

IRC

Eli Copter

ETI

Emma Carena, Jack Uzzi

CGP

Aude Javel

Les scripts placés dans /docker-entrypoint-initdb.d/ sont exécutés lors de la première initialisation d'un répertoire de données PostgreSQL vide. Ils ne sont pas relancés automatiquement à chaque redémarrage du conteneur.

Pour lancer localement l'image personnalisée depuis la racine du projet :

docker build -t mydatabase ./database
docker network create app-network
docker run -d \
  --name my-database \
  --network app-network \
  -e POSTGRES_DB=db \
  -e POSTGRES_USER=usr \
  -e POSTGRES_PASSWORD=pwd \
  -v postgres-data:/var/lib/postgresql/data \
  mydatabase

pwd est une valeur de démonstration du TP, pas un mot de passe à réutiliser en production. En environnement réel, utiliser un secret dédié. Le volume nommé permet de conserver les données lorsque le conteneur est supprimé ou recréé.

Pour vérifier les tables :

docker exec -it my-database psql -U usr -d db

Puis dans psql :

\dt
SELECT * FROM departments;
SELECT * FROM students;

Adminer peut également être utilisé comme interface d'administration locale, reliée au même réseau Docker :

docker run -d \
  --name adminer \
  --network app-network \
  -p 8090:8080 \
  adminer

2.2 Backend Spring Boot

Le backend est une API REST Java/Spring Boot avec Maven. Les sources sont dans backend/simpleapi/.

Le Dockerfile utilise une construction en plusieurs étapes (multistage build) : le JDK et Maven servent à compiler l'application, puis seul le JRE et le fichier JAR sont copiés dans l'image d'exécution. Cela évite d'embarquer les outils de compilation inutiles dans l'image finale.

Exemple de structure :

FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /opt/myapp
RUN apk add --no-cache maven
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /opt/myapp
COPY --from=build /opt/myapp/target/*.jar ./myapp.jar
ENTRYPOINT ["java", "-jar", "myapp.jar"]

Le backend communique avec PostgreSQL via le nom du conteneur/service sur le réseau Docker, par exemple database:5432. Dans le déploiement distant, la configuration Spring est fournie par variables d'environnement :

SPRING_DATASOURCE_URL=jdbc:postgresql://database:5432/db
SPRING_DATASOURCE_USERNAME=usr
SPRING_DATASOURCE_PASSWORD=<secret>

La configuration de la base et le mot de passe doivent correspondre à ceux utilisés par PostgreSQL.

Les principaux endpoints utilisés pour les vérifications sont :

GET /departments

GET /students

2.3 Frontend

Le frontend basé sur Vue.js est issu du dépôt https://github.com/takima-training/devops-front et est servi par Nginx. Il doit être construit avec une URL d'API qui cible le serveur et ses routes publiques. La configuration de production utilisée dans le TP suit cette forme :

VUE_APP_API_URL=kadjio.mballa.takima.school

L'application utilise cette valeur pour interroger les endpoints de l'API. Après toute modification de .env.production, il faut reconstruire et republier l'image frontend : les variables VUE_APP_* sont généralement intégrées lors de la compilation du frontend.

2.4 Apache HTTPD comme reverse proxy

Le conteneur HTTPD publie le port 80 et route les requêtes vers le bon service. Les routes de l'API doivent précéder la route générale /.

Extrait important de httpd/httpd.conf :

<VirtualHost *:80>
    ProxyPreserveHost On

    ProxyPass        /departments http://backend:8080/departments
    ProxyPassReverse /departments http://backend:8080/departments

    ProxyPass        /students http://backend:8080/students
    ProxyPassReverse /students http://backend:8080/students

    ProxyPass        / http://frontend:80/
    ProxyPassReverse / http://frontend:80/
</VirtualHost>

Le Dockerfile HTTPD copie la page et la configuration dans l'image :

FROM httpd:2.4
COPY index.html /usr/local/apache2/htdocs/index.html
COPY httpd.conf /usr/local/apache2/conf/httpd.conf

La configuration Apache doit activer les modules proxy nécessaires (mod_proxy et mod_proxy_http). Si / renvoie du JSON au lieu du frontend, vérifier que la configuration active ne route pas encore / vers backend:8080.

2.5 Réseaux, volumes et orchestration

Dans le déploiement final, les réseaux séparent les communications :

backend-network relie la base de données au backend.

frontend-network relie le backend, le frontend et le reverse proxy.

Le volume nommé postgres-data conserve les données PostgreSQL.

Docker Compose peut décrire les services, réseaux, volumes, variables d'environnement, dépendances et politiques de redémarrage dans un seul fichier docker-compose.yml. Les commandes principales sont :

docker compose up -d --build
docker compose ps
docker compose logs -f
docker compose restart
 docker compose down

La commande docker compose down -v supprime également les volumes déclarés : elle peut donc supprimer les données de la base. Ne pas l'utiliser sans avoir décidé explicitement d'effacer les données.

3. TP2 — Images et intégration continue avec GitHub Actions

3.1 Publication des images sur Docker Hub

Les images doivent être publiées dans un registre afin de pouvoir être téléchargées par le serveur distant.

Exemple de commandes manuelles :

docker login
docker tag mydatabase rudi2020/mydatabase:latest
docker push rudi2020/mydatabase:latest

Le même principe s'applique au backend, au frontend et à HTTPD. Le nom publié doit correspondre au nom d'image utilisé par Ansible.

3.2 Workflow GitHub Actions

Le workflow est placé dans .github/workflows/main.yml. Il est déclenché lors d'un push sur master ou develop, ainsi que sur les pull requests.

Les étapes prévues sont :

Récupérer le dépôt.

Installer Java 21 et lancer mvn clean verify pour tester le backend.

Construire les images Docker du backend, de la base de données (si une image personnalisée est utilisée), de HTTPD et du frontend.

Sur les pushes vers master, s'authentifier auprès de Docker Hub et publier les images.

Une fois les publications réussies, lancer Ansible pour mettre à jour le serveur distant.

Extrait de la commande de test du backend :

mvn clean verify --file ./backend/simpleapi/pom.xml

Le déploiement est conditionné aux jobs de construction/publication : il ne doit pas démarrer si un des jobs requis échoue. Les identifiants Docker Hub ne doivent pas être écrits en clair dans le YAML ; ils doivent être enregistrés dans les secrets du dépôt GitHub :

DOCKERHUB_USERNAME

DOCKERHUB_TOKEN

SSH_PRIVATE_KEY

Le secret SSH_PRIVATE_KEY sert à authentifier le runner GitHub Actions auprès du serveur. La clé privée ne doit jamais être commitée dans Git.

État de validation : la présence du workflow ne suffit pas à prouver que la livraison continue fonctionne. Le checkpoint est validé lorsqu'une exécution GitHub Actions complète est verte, que les images attendues sont présentes dans Docker Hub et que l'application reste accessible après le déploiement.

4. TP3 — Déploiement distant avec Ansible

4.1 Environnement

Le développement est réalisé sous Windows avec Docker Desktop et WSL2/Ubuntu. Ansible s'exécute dans WSL et se connecte au serveur Debian via SSH.

Hôte distant : kadjio.mballa.takima.school

Utilisateur SSH : admin

Inventaire : ansible/inventories/setup.yml

Playbook principal : ansible/playbook.yml

Interpréteur Python distant utilisé par les modules Docker : /opt/docker_venv/bin/python

Exemple de structure d'inventaire (la clé SSH peut être donnée en ligne de commande plutôt que codée en dur) :

all:
  vars:
    ansible_user: admin
    ansible_python_interpreter: /opt/docker_venv/bin/python
  children:
    prod:
      hosts:
        kadjio.mballa.takima.school:

4.2 Playbook principal

Le playbook organise le déploiement sous forme de rôles réutilisables :

---
- name: Deploy the multi-container application
  hosts: all
  become: true
  gather_facts: true
  roles:
    - docker
    - network
    - database
    - backend
    - frontend
    - httpd

Rôle

Responsabilité

docker

Installer/configurer Docker sur la machine distante.

network

Créer les réseaux Docker nécessaires.

database

Démarrer PostgreSQL et attacher son volume persistant.

backend

Télécharger et démarrer l'image Spring Boot, configurée pour se connecter à PostgreSQL.

frontend

Télécharger et démarrer le frontend Nginx.

httpd

Télécharger et démarrer Apache, publier le port 80 et configurer le reverse proxy.

Les tâches utilisent la collection Ansible community.docker. Sur le serveur, Docker permet aux conteneurs de se joindre par leurs noms sur les réseaux communs.

4.3 Exécuter et vérifier Ansible manuellement

Depuis WSL, ouvrir le dossier ansible :

cd /mnt/c/Users/mball/Desktop/EFREI/I2/docker/TP1/ansible

Tester la connectivité :

ansible all -i inventories/setup.yml -m ping

Valider la syntaxe du playbook :

ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check

Déployer :

ansible-playbook -i inventories/setup.yml playbook.yml

Si la clé SSH n'est pas déclarée dans l'inventaire :

ansible-playbook \
  -i inventories/setup.yml \
  playbook.yml \
  --private-key /home/mball/.ssh/takima_key

Le même playbook est appelé depuis GitHub Actions après configuration de la clé privée dans le secret SSH_PRIVATE_KEY.

4.4 Ansible Vault — sécurisation des variables

Ansible Vault permet de chiffrer les variables sensibles, par exemple les identifiants PostgreSQL. Pour créer un fichier de variables chiffré :

ansible-vault create group_vars/all/vault.yml

Les variables utilisées par les rôles doivent ensuite être chargées depuis ce fichier. Lors d'une exécution manuelle, Ansible peut demander le mot de passe avec --ask-vault-pass. Dans GitHub Actions, le mot de passe Vault doit être stocké comme secret et injecté à l'exécution, sans être écrit dans le dépôt.

Important : créer un fichier Vault ne suffit pas. Pour valider ce point, il faut vérifier que le playbook charge effectivement les variables chiffrées et qu'un déploiement fonctionne sans secrets en clair dans les fichiers suivis par Git.

5. Configuration et structure du projet

Structure logique attendue à la racine du dépôt :

.
├── .github/
│   └── workflows/
│       └── main.yml
├── backend/
│   └── simpleapi/
│       ├── Dockerfile
│       ├── pom.xml
│       └── src/
├── database/
│   ├── Dockerfile
│   └── init/
│       ├── CreateScheme.sql
│       └── InsertData.sql
├── frontend/
│   ├── Dockerfile
│   ├── default.conf
│   └── .env.production
├── httpd/
│   ├── Dockerfile
│   ├── httpd.conf
│   └── index.html
└── ansible/
    ├── inventories/
    │   └── setup.yml
    ├── playbook.yml
    └── roles/
        ├── docker/
        ├── network/
        ├── database/
        ├── backend/
        ├── frontend/
        └── httpd/

Le détail des fichiers peut varier selon les étapes retenues. Ne pas versionner les mots de passe, jetons, clés privées ou fichiers contenant des secrets. Ajouter les fichiers locaux sensibles à .gitignore.

6. Vérification de l'application déployée

Vérifier l'API

curl -i http://kadjio.mballa.takima.school/departments
curl -i http://kadjio.mballa.takima.school/students

Résultat attendu : statut HTTP 200 OK et réponse JSON. /departments doit contenir les trois départements IRC, ETI, CGP; /students doit retourner les étudiants insérés dans la base.

Vérifier le frontend

curl -i http://kadjio.mballa.takima.school/

Résultat attendu : statut HTTP 200 OK et une réponse HTML, et non le JSON du endpoint de démonstration du backend.

Vérifier les conteneurs et les logs

Depuis le serveur, ou en utilisant Ansible depuis WSL :

ansible all -i inventories/setup.yml -b -m command -a "docker ps"
ansible all -i inventories/setup.yml -b -m command -a "docker logs --tail 100 backend"
ansible all -i inventories/setup.yml -b -m command -a "docker logs --tail 100 database"
ansible all -i inventories/setup.yml -b -m command -a "docker exec httpd grep -nE 'ProxyPass|ProxyPassReverse' /usr/local/apache2/conf/httpd.conf"

Dans GitHub, ouvrir Actions et vérifier que les tests, les jobs de publication d'images et le job de déploiement ont réussi. Faire les tests curl après le déploiement : un pipeline vert seul ne garantit pas que la nouvelle version de l'application répond correctement.

7. Questions du TP Docker

1-1 — Pourquoi fournir les variables avec -e plutôt que les inscrire dans le Dockerfile ?

Cela sépare la configuration de l'image et du code. La même image peut être lancée dans plusieurs environnements avec des paramètres différents. Cela évite également d'inscrire directement des valeurs sensibles dans le Dockerfile et dans l'historique de construction. -e n'est toutefois pas un coffre-fort : pour les vrais secrets, utiliser un gestionnaire de secrets ou un mécanisme adapté.

1-2 — Pourquoi attacher un volume au conteneur PostgreSQL ?

Le cycle de vie d'un conteneur est indépendant des données persistantes. Un volume permet de conserver les fichiers de la base quand le conteneur est supprimé ou recréé. Il permet ainsi de mettre à jour ou redéployer PostgreSQL sans effacer automatiquement les données.

1-3 — Éléments essentiels du conteneur de base de données

L'image de base est postgres:17.2-alpine; les scripts SQL sont copiés vers /docker-entrypoint-initdb.d/; les paramètres POSTGRES_DB, POSTGRES_USER et POSTGRES_PASSWORD sont définis au lancement; le port 5432 reste sur le réseau interne; les données sont conservées dans un volume monté sur /var/lib/postgresql/data.

1-4 — Pourquoi utiliser un multistage build ?

La première étape utilise une image JDK et Maven pour compiler l'application et produire un JAR. La seconde utilise une image JRE plus légère et ne copie que le JAR à exécuter. L'image finale ne contient donc pas le compilateur ni les outils de construction inutiles à l'exécution.

1-5 — Pourquoi utiliser un reverse proxy ?

Le reverse proxy fournit un point d'entrée unique, route les URL vers les services internes appropriés, évite d'exposer tous les ports et facilite la configuration centralisée de fonctions comme TLS, les règles d'accès ou l'équilibrage de charge. Ici, HTTPD envoie les routes d'API vers Spring Boot et les autres routes vers le frontend.

1-6 — Pourquoi Docker Compose est-il important ?

Compose décrit plusieurs services, leurs réseaux, volumes, variables et dépendances dans un fichier déclaratif. Il évite d'avoir à taper et maintenir de longues commandes docker run pour chaque conteneur, et permet de démarrer l'application complète de façon reproductible.

1-7 — Commandes Docker Compose importantes

docker compose up -d --build : construire les images si nécessaire et lancer les services en arrière-plan.

docker compose ps : afficher l'état des services.

docker compose logs -f : suivre les journaux.

docker compose restart : redémarrer les services.

docker compose down : arrêter et supprimer conteneurs et réseaux créés par Compose.

docker compose down -v : même action en supprimant aussi les volumes déclarés; à éviter si les données doivent être conservées.

1-8 — Que contient docker-compose.yml ?

Il décrit les services database, backend, frontend et httpd, leurs images ou contextes de construction, variables d'environnement, réseaux, volumes, ports publiés, dépendances et politiques de redémarrage. Seul le reverse proxy doit publier le port HTTP public; la base et le backend sont accessibles via les réseaux Docker internes.

1-9 — Comment publier une image sur Docker Hub ?

docker login
docker tag <image-locale> <utilisateur>/<nom-image>:<tag>
docker push <utilisateur>/<nom-image>:<tag>

Il faut utiliser des tags identifiables. Le tag latest est pratique pour un TP, tandis qu'un tag associé au commit ou à une version facilite la traçabilité et le rollback.

1-10 — Pourquoi publier des images dans un registre en ligne ?

Un registre permet de partager les images entre développeurs, runners CI et serveurs distants. Le serveur peut télécharger exactement les images publiées au lieu de les reconstruire localement. Il centralise aussi les versions et leur distribution.

8. Sécurité et améliorations possibles

Déployer automatiquement toute nouvelle image portant le tag latest présente des risques : régression, vulnérabilité, image altérée ou déploiement non reproductible. Les améliorations possibles sont :

lancer les tests avant de publier les images;

déployer seulement à partir d'une branche protégée et de builds réussis;

utiliser des tags immuables fondés sur le commit ou des digests d'image;

scanner les images et vérifier leur provenance;

limiter les permissions des tokens Docker Hub et de la clé SSH;

garder les secrets dans GitHub Actions Secrets et Ansible Vault, jamais dans le dépôt;

mettre en place des health checks, une vérification après déploiement et une stratégie de rollback.

Les versions d'images et les dépendances devraient être maintenues et mises à jour régulièrement. Les secrets d'exemple du TP (usr, pwd) sont uniquement des valeurs de laboratoire.

9. Dépannage

Symptôme

Vérifications

Le site racine retourne du JSON

Vérifier le ProxyPass / actif dans le conteneur HTTPD; il doit pointer vers frontend:80. Vérifier que l'image HTTPD a bien été reconstruite, publiée, téléchargée et que le conteneur a été recréé.

/students renvoie HTTP 500

Vérifier les logs du backend et de la base; confirmer la présence des tables, les valeurs de connexion JDBC, les identifiants et le réseau Docker partagé.

PostgreSQL redémarre mais les scripts SQL ne se rejouent pas

Les scripts d'initialisation ne s'exécutent qu'à l'initialisation d'un répertoire de données vide. Le volume déjà initialisé est conservé; ne pas le supprimer sans sauvegarde et décision explicite.

GitHub Actions ne se connecte pas à Docker Hub

Vérifier les secrets DOCKERHUB_USERNAME et DOCKERHUB_TOKEN, ainsi que les permissions du token.

Le job Ansible échoue en SSH

Vérifier SSH_PRIVATE_KEY, le format de la clé, l'utilisateur admin, la résolution du nom d'hôte et l'accès réseau au serveur.

La nouvelle image semble ne pas être utilisée

Vérifier que le job a publié l'image attendue, que le rôle Ansible la télécharge puis met à jour le conteneur, et comparer les tags/digests exécutés.

Note de validation : les commandes ci-dessus permettent de démontrer le fonctionnement. Les checkpoints CI/CD et Vault doivent être déclarés réussis uniquement après une exécution réelle et réussie qui les vérifie.
