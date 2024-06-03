FROM php:8.3-fpm-alpine

# docker-compose cp php83:/usr/local/etc/php-fpm.d/www.conf ~/code/docker-develop/php83/www.conf
# docker-compose cp php83:/usr/local/etc/php/php.ini-development ~/code/docker-develop/php83/php.ini

COPY php.ini "$PHP_INI_DIR/php.ini"
COPY www.conf /usr/local/etc/php-fpm.d/www.conf

WORKDIR /www

RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.tuna.tsinghua.edu.cn/g' /etc/apk/repositories

RUN apk add --no-cache \
        $PHPIZE_DEPS \
        iputils 

# 安装扩展
# https://github.com/mlocati/docker-php-extension-installer
RUN docker-php-ext-install pdo_mysql

# # 安装 redis 扩展
RUN pecl install redis-6.0.2

# 开启扩展
RUN docker-php-ext-enable pdo_mysql redis



