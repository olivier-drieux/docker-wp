ARG WORDPRESS_IMAGE

# Load WordPress image
FROM ${WORDPRESS_IMAGE}

ARG SSH_USER
ARG SSH_PASSWORD
ARG ROOT_PASSWORD
ARG WORDPRESS_VERSION
ARG PHP_VERSION
ARG INSTALL_PATH

# Update the image and install required packages
RUN apt-get update \
    && apt-get upgrade -y \
    && apt-get install -y --no-install-recommends \
        sudo \
        nano \
        openssh-server \
        mariadb-server \
        less \
        curl

# Create new user
RUN useradd -s /bin/bash -g sudo -m -p $(openssl passwd -1 ${SSH_PASSWORD}) ${SSH_USER}

# Change root password
RUN echo "root:${ROOT_PASSWORD}" | chpasswd

# Configure SSH
RUN mkdir -p /run/sshd

# Download and install WP-CLI
RUN curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \
    && chmod +x wp-cli.phar \
    && mv wp-cli.phar /usr/local/bin/wp

# Download WordPress in shared folder
RUN mkdir -p ${INSTALL_PATH}/wordpress/${WORDPRESS_VERSION} \
    && cd ${INSTALL_PATH}/wordpress/${WORDPRESS_VERSION} \
    && wp core download --locale=fr_FR --version=${WORDPRESS_VERSION} --force --allow-root

# Copy templates
RUN mkdir ${INSTALL_PATH}/templates
COPY ./templates ${INSTALL_PATH}/templates
RUN chmod -R 755 ${INSTALL_PATH}/templates

# Copy script to start apache2 and sshd proccesses
COPY ./start.sh /start.sh
RUN chmod 755 /start.sh

EXPOSE 22 80
CMD ["/start.sh"]
