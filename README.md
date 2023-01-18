# Projet devOps Diginamic du 18/01/2023 au 20/01/2023

(docker compose) -> pas sûr; à cause du script de demande de choix CMS

- container mariadb -> volume nommé (protection pour que ça ne soit pas accessible de l'extérieur)
- container php-fpm -> volume d'hôte
- container apache -> volume d'hôte / COPY de php-fpm vers ici pour que php soit utilisable
- container pour sauvegardes : db et src code -> volume nommé (même que Mdb)

-> tous les containers sur le même réseau

check pour connection SSH
github
