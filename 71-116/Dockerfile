# build site dependencies from debian stretch php 7.3 image
FROM php:7.3-buster as builder

ENV NODE_VERSION 12.x

RUN apt-get update && apt-get install -y gnupg
RUN curl -sL https://deb.nodesource.com/setup_$NODE_VERSION | bash -
RUN apt-get update && apt-get install -y --no-install-recommends \
        libzip-dev \
        build-essential \
        libfreetype6-dev \
        libjpeg62-turbo-dev \
        libmcrypt-dev \
        libpng-dev \
        default-mysql-client \
        git \
        nodejs \
    && docker-php-ext-install \
        gd \
        opcache \
        pdo \
        pdo_mysql \
        zip

RUN rm -rf /var/lib/apt/lists/*

ENV COMPOSER_COMMIT 6e5e46fc7cab0015d5961f54bb39be1f4b330093
ENV COMPOSER_SHA384SUM a5c698ffe4b8e849a443b120cd5ba38043260d5c4023dbf93e1558871f1f07f58274fc6f4c93bcfd858c6bd0775cd8d1
RUN curl -o composer-setup.php "https://raw.githubusercontent.com/composer/getcomposer.org/$COMPOSER_COMMIT/web/installer" | php -- --quiet > 'composer-setup.php' \
    && php -r "if (hash_file('SHA384', 'composer-setup.php') === '$COMPOSER_SHA384SUM' ) { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;" \
    && php composer-setup.php --install-dir=/usr/local/bin --filename=composer \
    && php -r "unlink('composer-setup.php');"

WORKDIR /root

# build the server from debian stretch and nginx 1.16 stable
FROM php:7.3-fpm-buster

# non root user to run the server after binding sockets as root
ENV NGINX_DOC_ROOT /var/www/html/web
ENV NGINX_VERSION   1.16.1
ENV NJS_VERSION     0.3.5
ENV PKG_RELEASE     1~buster
ENV DRUSH_VERSION   0.6.0

RUN set -x \
    && apt-get update \
    && apt-get install --no-install-recommends --no-install-suggests -y gnupg1 ca-certificates \
    && \
    NGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62; \
    found=''; \
    for server in \
        ha.pool.sks-keyservers.net \
        hkp://keyserver.ubuntu.com:80 \
        hkp://p80.pool.sks-keyservers.net:80 \
        pgp.mit.edu \
    ; do \
        echo "Fetching GPG key $NGINX_GPGKEY from $server"; \
        apt-key adv --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$NGINX_GPGKEY" && found=yes && break; \
    done; \
    test -z "$found" && echo >&2 "error: failed to fetch GPG key $NGINX_GPGKEY" && exit 1; \
    apt-get remove --purge --auto-remove -y gnupg1 && rm -rf /var/lib/apt/lists/* \
    && dpkgArch="$(dpkg --print-architecture)" \
    && nginxPackages=" \
        nginx=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-xslt=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-geoip=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-image-filter=${NGINX_VERSION}-${PKG_RELEASE} \
        nginx-module-njs=${NGINX_VERSION}.${NJS_VERSION}-${PKG_RELEASE} \
    " \
    && case "$dpkgArch" in \
        amd64|i386) \
# arches officialy built by upstream
            echo "deb https://nginx.org/packages/debian/ buster nginx" >> /etc/apt/sources.list.d/nginx.list \
            && apt-get update \
            ;; \
        *) \
# we're on an architecture upstream doesn't officially build for
# let's build binaries from the published source packages
            echo "deb-src https://nginx.org/packages/debian/ buster nginx" >> /etc/apt/sources.list.d/nginx.list \
            \
# new directory for storing sources and .deb files
            && tempDir="$(mktemp -d)" \
            && chmod 777 "$tempDir" \
# (777 to ensure APT's "_apt" user can access it too)
            \
# save list of currently-installed packages so build dependencies can be cleanly removed later
            && savedAptMark="$(apt-mark showmanual)" \
            \
# build .deb files from upstream's source packages (which are verified by apt-get)
            && apt-get update \
            && apt-get build-dep -y $nginxPackages \
            && ( \
                cd "$tempDir" \
                && DEB_BUILD_OPTIONS="nocheck parallel=$(nproc)" \
                    apt-get source --compile $nginxPackages \
            ) \
# we don't remove APT lists here because they get re-downloaded and removed later
            \
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
# (which is done after we install the built packages so we don't have to redownload any overlapping dependencies)
            && apt-mark showmanual | xargs apt-mark auto > /dev/null \
            && { [ -z "$savedAptMark" ] || apt-mark manual $savedAptMark; } \
            \
# create a temporary local APT repo to install from (so that dependency resolution can be handled by APT, as it should be)
            && ls -lAFh "$tempDir" \
            && ( cd "$tempDir" && dpkg-scanpackages . > Packages ) \
            && grep '^Package: ' "$tempDir/Packages" \
            && echo "deb [ trusted=yes ] file://$tempDir ./" > /etc/apt/sources.list.d/temp.list \
# work around the following APT issue by using "Acquire::GzipIndexes=false" (overriding "/etc/apt/apt.conf.d/docker-gzip-indexes")
#   Could not open file /var/lib/apt/lists/partial/_tmp_tmp.ODWljpQfkE_._Packages - open (13: Permission denied)
#   ...
#   E: Failed to fetch store:/var/lib/apt/lists/partial/_tmp_tmp.ODWljpQfkE_._Packages  Could not open file /var/lib/apt/lists/partial/_tmp_tmp.ODWljpQfkE_._Packages - open (13: Permission denied)
            && apt-get -o Acquire::GzipIndexes=false update \
            ;; \
    esac \
    \
    && apt-get install --no-install-recommends --no-install-suggests -y \
                        $nginxPackages \
                        gettext-base \
    && apt-get remove --purge --auto-remove -y ca-certificates && rm -rf /var/lib/apt/lists/* /etc/apt/sources.list.d/nginx.list \
    \
# if we have leftovers from building, let's purge them (including extra, unnecessary build deps)
    && if [ -n "$tempDir" ]; then \
        apt-get purge -y --auto-remove \
        && rm -rf "$tempDir" /etc/apt/sources.list.d/temp.list; \
    fi

WORKDIR /var/www/html
RUN apt-get update && apt-get install -y --no-install-recommends \
    default-mysql-client \
    libpng-dev \
    libzip-dev \
    ca-certificates \
    supervisor \
    && docker-php-ext-install gd \
    opcache \
    pdo \
    pdo_mysql \
    zip \
    && rm -rf /var/lib/apt/lists/*


# Grab config dependencies for php, nginx and supervisor
RUN curl -O https://raw.githubusercontent.com/Bixal/php-nginx-drupal/master/71-116/php/php.ini
RUN curl -O https://raw.githubusercontent.com/Bixal/php-nginx-drupal/master/71-116/nginx.conf
RUN curl -O https://raw.githubusercontent.com/Bixal/php-nginx-drupal/master/71-116/supervisord.conf
RUN mv php.ini /usr/local/etc/php
RUN mv supervisord.conf /etc/
RUN mv nginx.conf /etc/nginx/

# Install latest version of drupal console
RUN curl https://drupalconsole.com/installer -L -o drupal.phar
RUN mv drupal.phar /usr/local/bin/drupal
RUN chmod +x /usr/local/bin/drupal

# Install Drush launcher to add drush command to $PATH
RUN curl -o drush.phar https://github.com/drush-ops/drush-launcher/releases/download/$DRUSH_VERSION/drush.phar
RUN mv drush.phar /usr/local/bin/drush
RUN chmod +x /usr/local/bin/drush


# copy in the files that the builder image built for drupal
COPY --from=builder /var/www/html /var/www/html

# make nginx able to access web root
RUN chown -R www-data:www-data /var/www/html

EXPOSE 80

STOPSIGNAL SIGTERM

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
