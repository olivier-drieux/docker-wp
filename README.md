# Website building Docker image

## Fonctionnalités
Ce projet contient le conteneur "wordpress-container" qui contient :
    - un serveur SSH
    - un serveur apache2
    - WP-CLI
    - WordPress
    - Des modèles WordPress

## Configuration
### Fichier de variable d'environnement
Le fichier de variable d'environnement `.env` contient les variables suivantes :
- `SSH_USER` : nom de l'utilisateur pour se connecter en SSH
- `SSH_PASSWORD` : mot de passe de l'utilisateur root
- `ROOT_PASSWORD` : mot de passe de l'utilisateur root
- `WORDPRESS_VERSION` : version de WordPress à installer
- `PHP_VERSION` : version de PHP à installer
- `INSTALL_PATH`: chemin vers le dossier où WordPress et les modèles seront téléchargés
- `WORDPRESS_DB_HOST` : hôte de la base de données
- `WORDPRESS_DB_USER` : nom d'utilisateur de la base de données
- `WORDPRESS_DB_PASSWORD` : mot de passe de l'utilisateur de la base de données
- `WORDPRESS_DB_NAME` : nom de la base de données
- `HTTP_PORT` : port d'écoute du serveur HTTP
- `SSH_PORT` : port d'écoute du serveur SSH

Exemple de fichier d'environnement :
```ini
###> SSH configuration ###
SSH_USER=user
SSH_PASSWORD=password
ROOT_PASSWORD=password
WORDPRESS_VERSION=6.0.2
PHP_VERSION=7.4
INSTALL_PATH=/home/shared
###< SSH configuration ###

###> Database configuration ###
WORDPRESS_DB_HOST=localhost:3306
WORDPRESS_DB_USER=root
WORDPRESS_DB_PASSWORD=password
WORDPRESS_DB_NAME=wordpress
###< Database configuration ###

###> Docker configuration ###
HTTP_PORT=8500
SSH_PORT=2222
###< Docker configuration ###
```
### Les modèles WordPress
Avant de déployer le conteneur, il est nécessaire d'installer les modèles WordPress dans le dossier `config/templates`.
En effet, les modèles sont déplacés pendant la construction du conteneur. Il faut reconstruire le conteneur pour ajouter un nouveau modèle.

## Installation
Pour lancer le conteneur, exécuter la commande suivante :
```sh
docker compose up -d
```
Docker va automatiquement télécharger les images nécessaires et créer le conteneur.

## Dockerfile
Ce Dockerfile reprend l'image WordPress (https://hub.docker.com/_/wordpress) suivant les arguments `WORDPRESS_VERSION` et `PHP_VERSION` sous la forme  `wordpress:${WORDPRESS_VERSION}-php${PHP_VERSION}-apache`.
Exemple :
```docker
FROM wordpress:6.0.2-php7.4-apache
```

Les paquets  `openssh-server` et `mariadb-server` sont téléchargés.
Le paquet `openssh-server` sert à créer le serveur SSH de l'image.
Le paquet `mariadb-server` permet à WP-CLI de communiquer avec la base de données.

L'utilisateur SSH est créé avec la commande suivante :
```docker
RUN useradd -s /bin/bash -g sudo -m -p $(openssl passwd -1 ${SSH_PASSWORD}) ${SSH_USER}
```
L'option `-s /bin/bash` permet de définir le shell par défaut de l'utilisateur.
L'option `-g sudo` permet d'ajouter le nouvel utilisateur au groupe `sudo` afin d'avoir accès aux fichiers et dossiers du groupe.
L'option `-m` permet de créer le dossier de l'utilisateur dans le dossier `/home`.
L'option `-p` permet renseigner le mot de passe du nouvel utilisateur avec l'argument `SSH_PASSWORD`.
Le nom de l'utilisateur est donné avec l'argument `SSH_USER`.

Le mot de passe root est modifié par l'argument `ROOT_PASSWORD` avec la commande suivante :
```docker
RUN echo "root:${ROOT_PASSWORD}" | chpasswd
```

WP-CLI est installé, comme indiqué dans la documentation (https://wp-cli.org/fr/#installation), avec la commande suivante :
```docker
RUN curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
	&& chmod +x wp-cli.phar \
	&& mv wp-cli.phar /usr/local/bin/wp
```

WordPress est téléchargé avec WP-CLI dans le dossier donné par l'argument `INSTALL_PATH`  avec la commande suivante :
```docker
RUN mkdir -p ${INSTALL_PATH}/wordpress/${WORDPRESS_VERSION} \
	&& cd ${INSTALL_PATH}/wordpress/${WORDPRESS_VERSION} \
	&& wp core download --locale=fr_FR --version=${WORDPRESS_VERSION} --force --allow-root
```
La version de WordPress est donné avec l'argument `WORDPRESS_VERSION`.
La commande `wp core download` est exécuté avec les paramètres suivants :
- `locale=fr_FR` : pour télécharger WordPress en français
- `version=${WORDPRESS_VERSION}` : pour télécharger la version de WordPress donné par l'argument `WORDPRESS_VERSION`
- `force` : écrase les fichiers existants dans le dossier
- `allow-root` : autorise l'exécution de la commande avec les droits root

Les modèles WordPress sont déplacés depuis le dossier `config/templates` vers le dossier du container donné par l'argument `INSTALL_PATH` avec la commande suivante :
```docker
RUN mkdir ${INSTALL_PATH}/templates
COPY ./templates ${INSTALL_PATH}/templates
RUN chmod -R 755 ${INSTALL_PATH}/templates
```
Les droits de lecture, écriture et exécution sont modifiés afin de permettre au processus de déploiement d'avoir les droits nécessaires sur les fichiers et dossiers des modèles.

Le fichier bash `start.sh` est déplacé depuis le dossier `config` vers le dossier `/` du container avec la commande suivante :
```docker
COPY ./start.sh /start.sh
RUN chmod 777 /start.sh
```
Ce fichier permet à docker de démarrer le serveur SSH en même temps que le serveur apache2 :
```bash
#!/bin/bash
/usr/sbin/sshd
/usr/sbin/apache2ctl -D FOREGROUND
```
L'option `-D FOREGROUND` indique au container de continuer son exécution tant que le processus `apache2` est en cours d'exécution.

Le Dockerfile se termine avec les 2 commandes suivantes :
```docker
EXPOSE 22 80
CMD ["/start.sh"]
```
La première commande permet d'ouvrir les ports `22` et `80` aux autres containers du réseau interne et la deuxième d'exécuter le fichier bash `start.sh`.
