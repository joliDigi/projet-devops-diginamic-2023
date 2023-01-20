# Projet devOps Diginamic du 18/01/2023 au 20/01/2023

## Sommaire

- [Énoncé](#énoncé)
- [Ébauche](#ébauche)
  - [Notes sur le travail d'ébauche](#notes-sur-le-travail-débauche)

<hr>

- [Les dockerfiles](#les-dockerfiles)
  - [Dockerfile install](#my_diginamic_projectwpfolder_for_dockerfileinstalldockerfile)
  - [Dockerfile backup](#my_diginamic_projectwpfolder_for_dockerfilebackupdockerfile)
- [Étapes avec docker compose](#étapes-avec-docker-compose)
- [Les scripts](#les-scripts)
  - [Le fichier de configuration apache](#le-fichier-de-configuration-apache)
  - [L'installation au choix d'un CMS](#linstallation-au-choix-dun-cms)
  - [L'installation de WordPress](#linstallation-de-wordpress)
  - [Le backup de la bdd](#le-backup-de-la-bdd)
  - [Notes sur les scripts](#notes-sur-les-scripts)

<hr>

- [Sources](#sources)
- [Extra : les étapes à la main](#extra--les-étapes-à-la-main)

## Énoncé

Vous devez mettre en place un serveur Web à partir de containers docker.

Il vous faut, un serveur MariaDb, un serveur type Apache et un interpréteur PHP.

Vous devez développer des scripts bah qui permettent de faire la sauvegarde de vos bases de données, la sauvegarde de vos codes sources.

Les sauvegardes doivent être placées dans un container distinct.

Vous devez développer un script bash qui permet de lancer au choix:

- l'installation de wordpress
- l'installation de Prestashop
- Les fichiers d configuration du serveur Web doivent être optimisés automatiquement en fonction de l'installation choisie

Vos containers doivent respecter les bonnes pratiques: volumes, réseau

Enfin, vous mettez en place un système qui devra permettre de se connecter aux containers via SSH.

## Ébauche

docker compose -> script de choix CMS sera dans le dockerfile de httpd ?? (vu qu'il vient en second)

- container mariadb
  - **à la main:** volume nommé (protection pour que ça ne soit pas accessible de l'extérieur)
  - **compose:** volume nommé interne
- container php-fpm -> volume d'hôte
- container apache -> volume d'hôte / COPY/VOLUME ?? de php-fpm vers ici pour que php soit utilisable
- container pour sauvegardes :
  - db et src code
  - volume nommé (même raison que Mariadb)
  - dockerfile qui lance un script qui
    - créer un crontab pour sauver et supprimer les + vieux régulièrement

-> tous les containers sur le même réseau

check pour connection SSH -> RESTE A FAIRE

github -> seulement pour le code source (j'imagine) -> RESTE A FAIRE

### **Arborescence des dossiers et fichiers**

/!\ CETTE PARTIE N'EST PLUS A JOUR LES DOSSIERS ONT CHANGES DE NOMS ET DE PLACE /!\

(2 versions pour les dockerfile -> n'en choisir qu'une évidement -> celle qui marchera)

- dossier sur l'hôte pour le code source (1)
- dossier projet sur WSL (1)
  - dockerfile (le dossier)
    - phpfpm
      - dockerfile (le fichier)
    - httpd
      - dockerfile (le fichier)
    - backup
      - dockerfile (le fichier)
  - scripts
    - backup.sh
    - ...
  - docker-compose.yml (mariadb est direct dedans)
  - Dockerfile-phpfpm (le fichier)
  - Dockerfile-httpd (le fichier)
  - Dockerfile-backup (le fichier)
- ??? les dossiers dans les conteneurs

### **Definition des chemins**

/!\ CETTE PARTIE N'EST PLUS A JOUR LES DOSSIERS ONT CHANGES DE NOMS ET DE PLACE /!\

- Hôte
  - C://Users/[username]/my_diginamic_project
- WSL
  - /my_diginamic_project/dockerfile/phpfpm/dockerfile
  - /my_diginamic_project/dockerfile/backup/dockerfile
  - /my_diginamic_project/scripts/backup.sh
  - /my_diginamic_project/dockerfile/httpd/dockerfile
  - /my_diginamic_project/Dockerfile-phpfpm
  - /my_diginamic_project/Dockerfile-httpd
  - /my_diginamic_project/Dockerfile-backup
- les conteneurs

<hr>

### Notes sur le travail d'ébauche

- J'aurais dû choisir les chemins à l'avance car je m'y' suis perdu. (J'en avais choisi à un endroit puis oublié, choisi d'autre ailleurs. Maintenant je dois tout relire)

## Les dockerfiles

### /my_diginamic_project/wp/folder_for_dockerfile/install/dockerfile

```dockerfile
FROM alpine:latest

WORKDIR /app

# de la machine vers le conteneur
COPY . .

CMD ["./install.sh"]
```

[Volume in dockerfile vs volume in docker-compose](https://stackoverflow.com/a/40568482)

### /my_diginamic_project/wp/folder_for_dockerfile/backup/dockerfile

```dockerfile
FROM alpine:latest

# set up of a daily task which runs the backup script
RUN echo "0 0 * * * ./my_diginamic_project/scripts/backup.sh" >> /var/spool/cron/crontabs/root
```

## Étapes avec docker compose

Pour WordPress

/my_diginamic_project/wp/docker-compose.yml

```yml
version: '2'

networks:
  my_docker_network:
    driver: bridge

volumes:
  my_db_volume:

services:
  my_phpfpm_container:
    image: bitnami/php-fpm:latest
    ports:
      - 9000:9000
    volumes:
      - .:/app
    networks:
      - my_docker_network

  my_apache_container:
    image: bitnami/apache:latest
    ports:
      - 80:8080
    volumes:
      - ./apache-vhost/myapp.conf:/vhosts/myapp.conf:ro
      - .:/app
    networks:
      - my_docker_network
    depends_on:
      - my_phpfpm_container

  my_alpine_container:
    volumes:
      - .:/app
    build:
      context: ./folder_for_dockerfile/alpine
    networks:
      - my_docker_network
    depends_on:
      - my_phpfpm_container
      - my_apache_container

  my_mariadb_container:
    image: mariadb:latest
    environment:
      - MARIADB_DATABASE=cms_db
      - MARIADB_USER=jolidigi
      - MARIADB_PASSWORD=mypassmot
      - MARIADB_ROOT_PASSWORD=admin
    volumes:
      - my_db_volume:/var/lib/mysql
    networks:
      - my_docker_network

  my_backups_container:
    build:
      context: ./folder_for_dockerfile/backup
    networks:
      - my_docker_network
    # Maybe a depends_on: - my_mariadb_container
```

Pour PrestaShop

/!\ CE FICHIER EST IDENTIQUE A CELUI DE WP /!\

/my_diginamic_project/ps/docker-compose.yml

```yml

```

## Les scripts

### Le fichier de configuration apache

/my_diginamic_project/apache-vhost/myapp.conf

```sh
LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
<VirtualHost *:8080>
  DocumentRoot "/app"
  ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php:9000/app/$1
  <Directory "/app">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
    DirectoryIndex index.php
  </Directory>
</VirtualHost>
```

### L'installation au choix d'un CMS

/my_diginamic_project/init_install.sh

```sh
#! /bin/bash

echo "Which CMS do you hate the least WordPress [wp] or PrestaShop [ps] ?"
while read choice; do
    if [ $choice = "wp" ]; then
        cd wp
        break
    fi
    if [ $choice = "ps" ]; then
        cd ps
        break
    fi
    echo -e "[wp] for WordPress\n[ps] for PrestaShop"
done

echo -e "You are now the unfortunate owner of a $choice project.\nI feel for you !"

docker compose up
```

### L'installation de WordPress

/my_diginamic_project/wp/install.sh

/my_diginamic_project/ps/install.sh

```sh
#! /bin/sh

apk update && apk upgrade
apk add unzip wget

wget https://wordpress.org/latest.zip
unzip latest.zip
rm latest.zip
```

### Le backup de la bdd

Script à lancer dans le conteneur de sauvegarde en tant que tâche crontab

/my_diginamic_project/scripts/backup.sh (à lancer dans une alpine)

```sh
#!/bin/sh

# This script dumps a backup of the database
# It saves the source code
# And potentially deletes old files

# Creates folders for the database and the source code
mkdir -p /my_backup/db
mkdir -p /my_backup/src_code

# today's date to the nanosecond to name backup files
DATE=$(date +%Y%m%d%H%M%S%N)

# Dump of the database
mysqldumps --host=my_mariadb_container --user=jolidigi --password=mariaP4$$w0rDdb cms_db > /my_backup/db/$DATE-backup.sql

# TODO save the src code

delete_old_files_and_log_deletion () {
  # Deletes all but the 20 newest files when it reaches 100 files
  if [ $(ls $1 | wc -w) -ge 100 ]; then
    for file in $(ls $1 | head -n -20); do
      rm "$1/$file"
    done
    # The following log wasn't asked of me but I thought it was a nice touch
    echo "Your backup has been cleaned up the $(date +%Y-%m-%d))" >>> "/my_backup/$2"
  fi
}

# For the database
delete_old_files_and_log /my_backup/db db_log
# For the source code
delete_old_files_and_log /my_backup/src_code src_code_log
```

### Notes sur les scripts

- Si je trouve le temps, liste de ce qui peut être améliorée:
  - mysql password: "Specifying a password on the command line should be considered insecure. You can use an option file to avoid giving the password on the command line." [mysql password](https://mariadb.com/kb/en/mariadb-dumpmysqldump/)

## Sources

Docker official sources:

- [alpine](https://hub.docker.com/_/alpine)
- [bitnami/php-fpm](https://hub.docker.com/r/bitnami/php-fpm/)
- [php:fpm](https://hub.docker.com/layers/library/php/fpm/images/sha256-cf2e0e1daf0ca2562c24155f212a1eac380698893d58eadde354b8d66ef10e0e?context=explore)
- [httpd](https://hub.docker.com/_/httpd)
- [mariadb](https://hub.docker.com/_/mariadb)
- [compose references](https://docs.docker.com/compose/compose-file/compose-file-v3)
- [compose getting started](https://docs.docker.com/compose/gettingstarted/)

<hr>

Other sources:

- [bitnami apache + fpm](https://docs.bitnami.com/tutorials/develop-http-api-php-containers/)
- [appache + fpm](https://www.bejean.eu/2020/11/18/apache-et-php-fpm-dans-des-conteneurs-separes)
- [mariadb dumps](https://mariadb.com/kb/en/mariadb-dumpmysqldump/)
- [dumps & crontab](https://www.kiloroot.com/mysql-database-backup-shell-scripts-that-can-be-run-as-cron-jobs/)
- [crontab alpine](https://mixu.wtf/cron-in-docker-alpine-image/)
- [crontab from digitalocean](https://www.digitalocean.com/community/tutorials/how-to-use-cron-to-automate-tasks-ubuntu-1804)

<hr>

Repository github:

- [mon repo projet](https://github.com/joliDigi/projet-devops-diginamic-2023)

## Extra : les étapes à la main

```sh
# création du réseau
docker network create my_docker_network

# création du volume nommé pour la bdd
docker volume create --name=my_db_volume
# création du volume nommé pour la sauvegarde
docker volume create --name=my_backup_volume

# création du conteneur mariadb
# By default, MariaDB will write its data files in /var/lib/mysql
docker run -d --name=my_mariadb_container --net=my_docker_network -v my_db_volume:/var/lib/mysql -e MARIADB_USER=jolidigi -e MARIADB_PASSWORD=mariaP4$$w0rDdb -e MARIADB_DATABASE=cms_db mariadb:latest


# construction de l'image php-fpm
docker build -t joliDigi/phpfpm:latest .
# création du conteneur php-fpm
docker run -dit --name=my_phpfpm_container --net=my_docker_network joliDigi/phpfpm


# construction de l'image apache
docker build -t joliDigi/httpd:latest .
# création du conteneur apache avec volume d'hôte
docker run -dit --name=my_httpd_container --net=my_docker_network -v C:\Users\joliDigi\my_diginamic_project:/usr/local/apache2/htdocs/ -p 8080:80 joliDigi/httpd

# création du conteneur de sauvegarde
docker run -dit --name=my_backups_container --net=my_docker_network -v my_backup_volume:/my_backup alpine
```
