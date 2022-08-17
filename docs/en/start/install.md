# Installation

You can install Spiral components independently or use a pre-build application skeleton, which enables most of the
framework functions. You can either downgrade Web skeleton or use the CLI skeleton with minimal number dependencies to
start a new application.

Default Web (full-stack) skeleton is available on https://github.com/spiral/app

Installation includes the following packages out of the box and framework-specific functions enabled:
- [roadrunner-bridge](https://github.com/spiral/roadrunner-bridge)
- [cycle-bridge](https://github.com/spiral/cylce-bridge)
- [spiral/nyholm-bridge](https://github.com/spiral/nyholm-bridge)

<br/>

Server Requirements
--------
Make sure that your server configured with the following PHP version and extensions:

* PHP 8.1+, 64bit
* *mb-string* extension (spiral is UTF-8 centric framework)

Web Application Bundle
--------
Application bundle includes the following components:

* High-performance HTTP, HTTP/2 server based on [RoadRunner](https://roadrunner.dev)
  via [roadrunner-bridge](https://github.com/spiral/roadrunner-bridge) package.
* Console commands via Symfony/Console
* Queue support for AMQP, Beanstalk, Amazon SQS, in-Memory
* Stempler template engine
* Translation support by Symfony/Translation
* Security, validation, filter models
* PSR-7 HTTP pipeline, session, encrypted cookies
* DBAL and migrations support
* Monolog, Dotenv
* Prometheus metrics
* [Cycle DataMapper ORM](https://github.com/cycle) integration
  via [cycle-bridge](https://github.com/spiral/cycle-bridge) package.

Installation
--------

```bash
composer create-project spiral/app
```

> **Note**
> Application server will be downloaded automatically (`php-curl` and `php-zip` required).

Once the application installed, you can ensure that it was configured properly by executing:

```bash
php app.php configure
```

To start application server execute:

```bash
./rr serve
```

On Windows:

```bash
./rr.exe serve
```

The application will be available on `http://localhost:8080`.

> **Note**
> Read more about application server configuration [here](https://roadrunner.dev/docs).

## Other Skeletons

Check other skeleton builds:

- https://github.com/spiral/app-cli - minimal CLI build
- https://github.com/spiral/app-grpc - GRPC specific build (no views, HTTP)
