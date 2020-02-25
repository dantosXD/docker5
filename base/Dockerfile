ARG CCM_PHP_VERSION=7.2

FROM php:${CCM_PHP_VERSION}-fpm

ENV C5_SUDOAS=www-data

ARG DEBIAN_FRONTEND=noninteractive

ARG PHP_EXTENSIONS="bcmath bz2 curl exif ftp gd gmp imagick imap intl json ldap mbstring opcache pdo pdo_mysql redis soap ssh2 xml xsl zip"

COPY --from=mlocati/php-extension-installer /usr/bin/install-php-extensions /usr/local/bin/
COPY assets/ccm-service /usr/local/bin/
COPY assets/entrypoint.sh /
COPY assets/bashrc /root/.bashrc

ADD https://raw.githubusercontent.com/concrete5/cli/master/c5 /usr/local/bin/
ADD https://raw.githubusercontent.com/concrete5/cli/master/composerpkg /usr/local/bin/

EXPOSE 80
EXPOSE 3306
EXPOSE 9000

RUN \
    echo '### Updating system' && \
    apt-get update -qq && \
    apt-get upgrade -yqq && \
    apt-get autoremove --purge -yqq && \
    apt-get install -yqq gnupg unzip sudo apt-transport-https git && \
    \
    echo '### Configuring helper commands' && \
    chmod 0755 \
        /usr/local/bin/ccm-service \
        /usr/local/bin/c5 \
        /usr/local/bin/composerpkg \
        /entrypoint.sh \
        && \
    \
    echo '### Installing MariaDB' && \
    curl -sSL https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | bash -s -- --skip-maxscale && \
    apt-get install -yqq mariadb-server && \
    \
    echo '### Installing Nginx' && \
    curl -sSL -o nginx_signing.key https://nginx.org/keys/nginx_signing.key && \
    APT_KEY_DONT_WARN_ON_DANGEROUS_USAGE=1 apt-key add nginx_signing.key && \
    rm nginx_signing.key && \
    echo "deb https://nginx.org/packages/debian/ $(. /etc/os-release; printf '%s' "$VERSION" | cut -d\( -f2 | cut -d\) -f1) nginx" >/etc/apt/sources.list.d/nginx.list && \
    apt-get update -yqq && \
    apt-get install -yqq nginx && \
    \
    echo '### Configuring PHP' && \
    install-php-extensions $PHP_EXTENSIONS && \
    echo 'pdo_mysql.default_socket=/var/run/mysqld/mysqld.sock' >>/usr/local/etc/php/conf.d/docker-php-ext-pdo_mysql.ini && \
    \
    echo '### Installing Composer' && \
    curl -sSL https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer && \
    mkdir -p /var/www/.composer && \
    chown -R www-data:www-data /var/www/.composer && \
    (sudo -u www-data -- composer global require hirak/prestissimo || true) && \
    (sudo -u www-data -- composer clear-cache || true) && \
    \
    echo '### Configuring database' && \
    sed -i -r 's/^(\s*bind-address\s*=\s*)127\.0\.0\.1(\s*)$/\10.0.0.0\2/' /etc/mysql/my.cnf && \
    sed -i -r 's/^(\s*innodb_flush_method\s*=\s*)\S.*?(\s*)$/\1fsync\2/' /etc/mysql/my.cnf && \
    ccm-service start db && \
    echo "CREATE USER 'c5'@'%' IDENTIFIED BY '12345'; CREATE DATABASE c5 COLLATE 'utf8mb4_unicode_ci'; GRANT ALL PRIVILEGES ON c5.* TO 'c5'@'%'; FLUSH PRIVILEGES;" | mysql && \
    \
    echo '### Final operations' && \
    mkdir /app && \
    chown www-data:www-data /app && \
    ccm-service stop && \
    \
    echo 'Ready.'

COPY assets/nginx.conf /etc/nginx/conf.d/default.conf
COPY assets/php-fpm.conf /usr/local/etc/php-fpm.d/custom.conf

WORKDIR /app

ENTRYPOINT [ "/entrypoint.sh" ]