# PHP Nginx Drupal Docker image
Docker image created to run both nginx and php-fpm in one container under
[supervisor](http://supervisord.org/index.html). This helps in cases when only
one docker container is to be used, like single container AWS Elastic Beanstalk.

## Included
Image base - Debian Buster
Nginx - 1.6.1
Node.js - 12.x
Composer - 1.9.1
Drupal Console - latest stable
Drush - latest stable

## Installing
Available on [Dockerhub](https://hub.docker.com/r/bixal/php-nginx-drupal) but
you will also need the php/php.ini, nginx.conf and supervisord.conf files.

## Logs
Sent to STDOUT for docker to capture.
