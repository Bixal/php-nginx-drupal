# PHP Nginx Drupal Docker image
Docker image created to run both nginx and php-fpm in one container under
[supervisor](http://supervisord.org/index.html). This helps in cases when only
one docker container is to be used, like single container AWS Elastic Beanstalk.

## Included
Image base - Debian Buster

Nginx - 1.18.0

Composer - latest

Drush - latest

## Installing
Available on [Dockerhub](https://hub.docker.com/r/bixal/php-nginx-drupal).

## Logs
Sent to STDOUT for docker to capture.
