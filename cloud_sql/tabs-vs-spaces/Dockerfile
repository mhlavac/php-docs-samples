FROM php:7.3-stretch
EXPOSE 8080

RUN docker-php-ext-install pdo_mysql

COPY . /usr/src/tabs-vs-spaces
WORKDIR /usr/src/tabs-vs-spaces

CMD [ "php", "-S", "0.0.0.0:8080", "index.php" ]

