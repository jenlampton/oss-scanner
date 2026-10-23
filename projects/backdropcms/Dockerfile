# from https://backdropcms.org/requirements
FROM php:8.3-apache

ENV DEBIAN_FRONTEND=noninteractive

# PHP extensions, git for the checkouts, and MariaDB
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git curl procps mariadb-server mariadb-client \
      libzip-dev libonig-dev libpng-dev libjpeg-dev libpq-dev \
  && docker-php-ext-configure gd --with-jpeg=/usr \
  && docker-php-ext-install gd mbstring pdo pdo_mysql pdo_pgsql zip \
  && rm -rf /var/lib/apt/lists/*

# Apache serves /var/www/html with .htaccess enabled
RUN a2enmod rewrite headers expires \
  && printf '<Directory /var/www/html>\n  Options -MultiViews +FollowSymLinks\n  AllowOverride All\n  Require all granted\n</Directory>\nServerName 127.0.0.1\n' \
       > /etc/apache2/conf-available/backdrop.conf \
  && a2enconf backdrop

# backdrop-start starts MariaDB and Apache.
RUN printf '%s\n' '#!/bin/sh' 'set -e' \
      'mkdir -p /run/mysqld && chown mysql:mysql /run/mysqld' \
      'mariadb-admin ping --silent 2>/dev/null || (mysqld_safe >/dev/null 2>&1 &)' \
      'until mariadb-admin ping --silent 2>/dev/null; do sleep 0.5; done' \
      'pgrep -x apache2 >/dev/null || apachectl start' \
      > /usr/local/bin/backdrop-start \
  && printf '%s\n' '#!/bin/sh' 'apachectl stop 2>/dev/null; mariadb-admin shutdown 2>/dev/null; true' \
      > /usr/local/bin/backdrop-stop \
  && chmod +x /usr/local/bin/backdrop-start /usr/local/bin/backdrop-stop

WORKDIR /var/www/html

# Clone Backdrop: https://github.com/backdrop/backdrop
ARG BACKDROP_BRANCH=1.x

RUN git clone --depth 1 --branch "${BACKDROP_BRANCH}" \
    https://github.com/backdrop/backdrop.git . \
  && chown -R www-data:www-data sites files

# Install Bee (Backdrop CLI): https://backdropcms.org/project/bee
# Leave BEE_REF empty to use the repo's default branch, or set it to a tag/branch to pin.
ARG BEE_REF=""

RUN git clone --depth 1 ${BEE_REF:+--branch "$BEE_REF"} \
    https://github.com/backdrop-contrib/bee.git /opt/bee \
  && rm -rf /opt/bee/.git

# bee runs as www-data (the user Apache runs as) from the Backdrop root, so the site's files stay writable by both.
RUN printf '#!/bin/sh\ncd /var/www/html && exec setpriv --reuid=www-data --regid=www-data --init-groups env HOME=/tmp php /opt/bee/bee.php "$@"\n' \
      > /usr/local/bin/bee \
  && chmod +x /usr/local/bin/bee

# Install a working site on MariaDB:
#   admin / admin

# Smoke-test the site
RUN backdrop-start \
  && curl -fsS -o /dev/null http://127.0.0.1/user/login \
  && test "$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1/core/modules/system/system.module)" = 403 \
  && backdrop-stop

# Start MariaDB and Apache, then stay in the foreground; stop both cleanly on `docker stop`.
CMD ["sh", "-c", "backdrop-start || exit 1; trap 'backdrop-stop; exit 0' TERM INT; sleep infinity & wait $!"]
