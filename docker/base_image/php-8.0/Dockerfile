FROM spiralscout/roadrunner:2.10.6 as roadrunner
FROM composer:2.3.9 as composer
FROM mlocati/php-extension-installer:1.5.29 as php-ext-installer
FROM php:8.0-bullseye as runtime
ARG GITHUB_PERSONAL_TOKEN
LABEL org.opencontainers.image.source=https://github.com/jikan-me/jikan-rest/docker/base_image/php-8.0
COPY --from=composer /usr/bin/composer /usr/bin/composer
COPY --from=php-ext-installer /usr/bin/install-php-extensions /usr/local/bin/
ENV COMPOSER_HOME="/tmp/composer"
RUN install-php-extensions gd exif intl bz2 gettext mongodb-stable redis opcache sockets pcntl

# install roadrunner
COPY --from=roadrunner /usr/bin/rr /usr/bin/rr
