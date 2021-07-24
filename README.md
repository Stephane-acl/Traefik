# Traefik pour docker

## Installation de traefik sur le serveur
* Prérequis :
    - Installer Docker et Docker-compose sur le serveur

* Première étape :
    - Créer un dossier traefik : `mkdir traefik`, puis aller dans ce dossier.
    - Installez l'utilitaire, qui est inclus dans le paquetage apache2-utils : `sudo apt-get install apache2-utils`.
    - Générer un mot de passe et un nom d'admin pour accéder au tableau de bord de traefik : `htpasswd -nb admin secure_password` (Exemple de la sortie : admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/)
    - On va utiliser cette sortie dans le fichier de configuration de Traefik pour configurer l'authentification HTTP de base pour le tableau de bord de contrôle et de surveillance de Traefik. 
    Copiez la ligne de sortie entière pour pouvoir la coller plus tard.
    - Pour configurer le serveur Traefik, vous allez créer deux nouveaux fichiers de configuration appelés `traefik.toml et traefik_dynamic.toml`. 
    - Ces fichiers vont nous permettent de configurer le serveur Traefik et les diverses intégrations, ou fournisseurs. 
    On va utiliser trois des fournisseurs disponibles de Traefik : api, docker, et acme. 
    Le dernier d'entre eux, acme, supporte les certificats TLS en utilisant Let's Encrypt.
    - Créer un fichier `traefik.toml` : `nano traefik.toml`.
    - Ajouter les configurations suivante :
        ```
        [entryPoints]
          [entryPoints.web]
            address = ":80"
            [entryPoints.web.http.redirections.entryPoint]
              to = "websecure"
              scheme = "https"
        
          [entryPoints.websecure]
            address = ":443"
        
        [api]
          dashboard = true
        
        [certificatesResolvers.lets-encrypt.acme]
          email = "compte@netime.fr"
          storage = "acme.json"
          [certificatesResolvers.lets-encrypt.acme.tlsChallenge]
        
        [providers.docker]
          watch = true
          network = "web"
        
        [providers.file]
          filename = "traefik_dynamic.toml"
         ```
    - Créer un fichier `traefik_dynamic.toml` : `nano traefik_dynamic.toml`.
    - Ajouter les configurations suivantes :
        ```
        [http.middlewares.simpleAuth.basicAuth]
          users = [
            "admin:$apr1$ruca84Hq$mbjdMZBAG.KWn7vfN/SNK/"
          ]

        [http.routers.api]
          rule = "Host(`mon-domain.com`)"
          entrypoints = ["websecure"]
          middlewares = ["simpleAuth"]
          service = "api@internal"
          [http.routers.api.tls]
            certResolver = "lets-encrypt"

        ```
* Deuxième étape
    - Dans cette étape, on va créer un réseau Docker pour que le proxy partage avec les conteneurs.
    - Créer un nouveau réseau appeler `web` par exemple : `docker network create web`
    Lorsque le conteneur Traefik démarre, on l'ajouteras à ce réseau. Plus tard, on pourra ajouter d'autres conteneurs à ce réseau pour que Traefik puisse s'y connecter.
    - Ensuite, créez un fichier vide qui contiendra vos informations Let's Encrypt. Vous le partagerez avec le conteneur pour que Traefik puisse l'utiliser : `touch acme.json`.
    - Traefik ne sera capable d'utiliser ce fichier que si l'utilisateur root à l'intérieur du conteneur a un accès unique en lecture et en écriture sur ce fichier. 
    Pour ce faire, verrouillez les permissions sur acme.json afin que seul le propriétaire du fichier ait les droits de lecture et d'écriture : `chmod 600 acme.json`.
    - Enfin, créez le conteneur Traefik avec cette commande : 
       ```
       docker run -d \
         -v /var/run/docker.sock:/var/run/docker.sock \
         -v $PWD/traefik.toml:/traefik.toml \
         -v $PWD/traefik_dynamic.toml:/traefik_dynamic.toml \
         -v $PWD/acme.json:/acme.json \
         -p 80:80 \
         -p 443:443 \
         --network web \
         --name traefik \
         traefik:v2.2
       ```
    - Une fois le conteneur lancé, on dispose maintenant d'un tableau de bord auquel on peut accéder pour voir l'état de santé de nos conteneurs : `https://monitor.your_domain/dashboard/`.
    - On sera invité à entrer notre nom d'utilisateur et notre mot de passe, qui sont le nom d'utilisateur et le mot de passe qu'on a configuré à l'étape 1.
    
## Démarrer et arrêter Traefik sur le serveur
* Pour démarrer le conteneur traefik : ```docker start traefik```
* Pour arrêter le conteneur traefik : ```docker stop traefik```

## Commandes utiles
* Supprimer les conteneurs, networks, volumes et images non utilisés :  ```docker system prune -a -f```
* Pour IMPORTER un fichier .sql utiliser la commande ```cat dump/symfony_app_user.sql | docker exec -i mysql-container mysql -uuser -ppassword db_name --default-character-set=utf8```
* Pour EXPORTER un fichier .sql utiliser ```docker exec CONTAINER /usr/bin/mysqldump -u root --password=root DATABASE > backup.sql --no-tablespaces```

## Ajouter un site sous docker en le connectant à traefik
   - Une fois le conteneur Traefik lancé, on est prêt à exécuter des applications derrière lui.

* Dans la config docker du projet créer un fichier `docker-compose.yml` : `nano docker-compose.yml`.

* Ajouter le network : 

    ```yaml
    networks:
        web:
          external: true
        internal:    
          external: false
    ```

* Ajouter les labels de traefik :
- Attention bien nommer son conteneur principale avec le nom du site et ne pas oublier de l'ajouter dans les labels.

    ```yaml
    services:
        monsiteweb:
            ...
         labels:
            - traefik.http.routers.monsiteweb.rule=Host(`monsiteweb.your_domain`)
            - traefik.http.routers.monsiteweb.tls=true
            - traefik.http.routers.monsiteweb.tls.certresolver=lets-encrypt
            - traefik.port=80
         networks:
            - internal
            - web
         depends_on:
            - mysql
    ```
 * Si bdd :

    ```yaml
      mysql:
        image: mysql:5.7
        environment:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_USER: username
          MYSQL_PASSWORD: password
          MYSQL_DATABASE: databasename
        networks:
          - internal
        labels:
          - traefik.enable=false
    ```
