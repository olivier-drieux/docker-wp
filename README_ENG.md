# Website building Docker image

## Features
This project contains the "wordpress-container" which contains :
    - an SSH server
    - an apache2 server
    - WP-CLI
    - WordPress
    - WordPress templates

## Configuration
### Environment variable file
The environment variable file `.env` contains the following variables:
- `SSH_USER` : user name to connect via SSH
- `SSH_PASSWORD` : password of the root user
- `ROOT_PASSWORD` : password of the root user
- `WORDPRESS_VERSION` : version of WordPress to install
- `PHP_VERSION`: version of PHP to install
- `INSTALL_PATH`: path to the folder where WordPress and templates will be downloaded
- `WORDPRESS_DB_HOST`: Database host
- `WORDPRESS_DB_USER`: database username
- `WORDPRESS_DB_PASSWORD`: database user's password
- `WORDPRESS_DB_NAME`: database name
- `HTTP_PORT`: HTTP server listening port
- `SSH_PORT` : SSH server listening port

Example of an environment file :
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
### WordPress templates
Before deploying the container, it is necessary to install the WordPress templates in the `config/templates` folder.
This is because the templates are moved around during the build of the container. You need to rebuild the container to add a new template.

## Installation
To launch the container, run the following command:
```sh
docker compose up -d
```
Docker will automatically download the necessary images and create the container.

## Dockerfile
This Dockerfile takes the WordPress image (https://hub.docker.com/_/wordpress) following the `WORDPRESS_VERSION` and `PHP_VERSION` arguments in the form `wordpress:${WORDPRESS_VERSION}-php${PHP_VERSION}-apache`.
Example:
```docker
FROM wordpress:6.0.2-php7.4-apache
```

The `openssh-server` and `mariadb-server` packages are downloaded.
The `openssh-server` package is used to create the SSH server for the image.
The `mariadb-server` package allows WP-CLI to communicate with the database.

The SSH user is created with the following command:
```docker
RUN useradd -s /bin/bash -g sudo -m -p $(openssl passwd -1 ${SSH_PASSWORD}) ${SSH_USER}
```
The `-s /bin/bash` option sets the default shell for the user.
The `-g sudo` option adds the new user to the `sudo` group so that they can access the group's files and folders.
The `-m` option creates the user's folder in the `/home` folder.
The `-p` option allows the password of the new user to be filled in with the `SSH_PASSWORD` argument.
The user's name is given with the `SSH_USER` argument.

The root password is changed by the `ROOT_PASSWORD` argument with the following command:
```docker
RUN echo "root:${ROOT_PASSWORD}" | chpasswd
```

WP-CLI is installed, as indicated in the documentation (https://wp-cli.org/fr/#installation), with the following command:
```docker
RUN curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
	&& chmod +x wp-cli.phar \
	&& mv wp-cli.phar /usr/local/bin/wp
```

WordPress is downloaded with WP-CLI to the folder given by the `INSTALL_PATH` argument with the following command:
```docker
RUN mkdir -p ${INSTALL_PATH}/wordpress/${WORDPRESS_VERSION} \
	&& cd ${INSTALL_PATH}/wordpress/${WORDPRESS_VERSION} \
	&& wp core download --locale=fr_FR --version=${WORDPRESS_VERSION} --force --allow-root
```
The version of WordPress is given with the `WORDPRESS_VERSION` argument.
The `wp core download` command is run with the following parameters:
- `locale=fr_FR`: to download WordPress in French
- `version=${WORDPRESS_VERSION}`: to download the version of WordPress given by the `WORDPRESS_VERSION` argument
- `force`: overwrites existing files in the folder
- `allow-root` : allow the command to be executed with root rights

WordPress templates are moved from the `config/templates` folder to the container folder given by the `INSTALL_PATH` argument with the following command:
```docker
RUN mkdir ${INSTALL_PATH}/templates
COPY ./templates ${INSTALL_PATH}/templates
RUN chmod -R 755 ${INSTALL_PATH}/templates
```
Read, write and execute permissions are modified to allow the deployment process to have the necessary permissions on template files and folders.

The bash file `start.sh` is moved from the `config` folder to the `/` folder of the container with the following command:
```docker
COPY ./start.sh /start.sh
RUN chmod 777 /start.sh
```
This file allows docker to start the SSH server at the same time as the apache2 server:
```bash
#!/bin/bash
/usr/sbin/sshd
/usr/sbin/apache2ctl -D FOREGROUND
```
The `-D FOREGROUND` option tells the container to continue running as long as the `apache2` process is running.

The Dockerfile is terminated with the following two commands:
```docker
EXPOSE 22 80
CMD ["/start.sh"]
```
The first command opens ports `22` and `80` to other containers on the internal network and the second executes the bash file `start.sh`.
