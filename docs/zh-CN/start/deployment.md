# 开始 - 部署

部署一个 Spiral 应用可能是一项复杂的任务，因为它需要执行多个步骤以确保应用程序正确配置并准备好用于生产环境。在本指南中，我们将探讨几种部署策略，包括基本的文件传输、使用源代码控制、构建脚本和 Docker 构建。

## 准备应用程序

在生产服务器上部署 Spiral 应用程序需要几个关键配置，以确保正确的操作和安全性。

> **警告**
> 不要将 `.env` 文件存储在你的代码仓库中，因为它可能包含敏感信息，例如数据库凭据、API 密钥和其他机密信息。

以下是在生产服务器上配置 Spiral 应用程序的逐步指南：

### 禁用调试模式

确保在应用程序的环境配置中将 `DEBUG` 变量设置为 `false`。这将防止在发生错误时显示敏感信息。

```dotenv .env
DEBUG=false
```

### 将环境设置为生产环境

将应用程序环境配置中的 `APP_ENV` 变量更改为 `production`。这将防止执行仅应在开发服务器上运行的意外操作。

```dotenv .env
APP_ENV=production
```

### 启用分词器缓存

确保在应用程序的环境配置中将 `TOKENIZER_CACHE_TARGETS` 变量设置为 `true`。这将禁用对应用程序源代码的扫描和分析，从而提高启动时的性能。

```dotenv .env
TOKENIZER_CACHE_TARGETS=true
```

> **注意**
> 在 [组件 - 静态分析](../advanced/tokenizer.md) 部分阅读有关分词器的更多信息。

### 设置详细级别

将 `VERBOSITY_LEVEL` 变量设置为 `basic` 级别，以隐藏服务器错误，防止向公众显示。

```dotenv .env
VERBOSITY_LEVEL=basic
```

### 配置日志记录器

在应用程序的环境配置中设置所需的 `MONOLOG_DEFAULT_CHANNEL`。这将确定应用程序日志的存储位置。同时，在应用程序的环境配置中将 `MONOLOG_DEFAULT_LEVEL` 变量设置为 `error`。这将防止记录调试和信息错误，有助于保持日志的清洁和易于阅读。

```dotenv .env
MONOLOG_DEFAULT_CHANNEL=roadrunner
MONOLOG_DEFAULT_LEVEL=error
```

### Cycle ORM

如果你正在使用 Cycle ORM，请确保在你的应用程序环境中将 `CYCLE_SCHEMA_CACHE` 和 `CYCLE_SCHEMA_WARMUP` 变量设置为 `true`。

`CYCLE_SCHEMA_CACHE` 变量控制 ORM 是否应该缓存数据库表的模式。当设置为 `true` 时，ORM 将缓存模式，这可以通过减少检索模式信息所需的数据库查询次数来提高应用程序的性能。

`CYCLE_SCHEMA_WARMUP` 变量控制 ORM 是否应该在应用程序启动时预热模式缓存。当设置为 `true` 时，ORM 将用模式信息预填充模式缓存，这可以进一步提高应用程序的性能，方法是减少检索模式信息所需的时间。

```dotenv .env
CYCLE_SCHEMA_CACHE=true
CYCLE_SCHEMA_WARMUP=true
```

### Composer

以下 composer 命令可用于为 Spiral 应用程序安装 vendor 包：

```terminal
composer install --optimize-autoloader --no-dev --no-scripts
```

此命令安装应用程序的依赖项，为应用程序生成自动加载器文件，并对其进行优化以提高性能。

-  `--no-dev` 选项用于防止安装任何开发依赖项
-  `--no-scripts` 选项用于防止执行任何 `post-install` 脚本。

## Nginx

### 代理到 RoadRunner

如果你想将 Spiral 与 RoadRunner 一起使用，并在其前面放置 Nginx 服务器，则需要将 Nginx 配置为反向代理。

以下是设置 Nginx 作为反向代理的示例。

```nginx /etc/nginx/sites-enabled/roadrunner.conf
server {
    listen 80;

    server_name _;

    location / {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

此配置指示 Nginx 监听端口 `80` 并将所有传入的请求转发到 `127.0.0.1:8080`，RoadRunner 在该端口运行。

你可以将此配置文件放在 `/etc/nginx/sites-available/` 目录中，然后创建一个指向它的符号链接，放在 `/etc/nginx/sites-enabled/` 目录中。

以下是使用 RoadRunner HTTP 服务器的配置示例：

```yaml .rr.yaml
http:
  address: 127.0.0.1:8080
```

> **警告：**
> 避免在 RoadRunner 配置中使用地址： `0.0.0.0:8080`，因为它会阻止直接访问 RoadRunner HTTP 服务器，仅允许通过 Nginx 反向代理进行访问。

### PHP-FPM

要将 Spiral 与 PHP-FPM 一起使用，你需要执行以下步骤：

1. 安装 `spiral/sapi-bridge` 包，该包提供一个 PHP-FPM 将用于处理请求的分发器。

使用以下命令安装该包：

```terminal
composer require spiral/sapi-bridge
```

安装该包后，你需要在应用程序的引导加载器列表中注册 `Spiral\Sapi\Bootloader\SapiBootloader` 引导加载器。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Sapi\Bootloader\SapiBootloader::class,
        // ...
    ];
}
```

在 [框架 - 引导加载器](../framework/bootloaders.md) 部分阅读有关引导加载器的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Sapi\Bootloader\SapiBootloader::class,
    // ...
];
```

在 [框架 - 引导加载器](../framework/bootloaders.md) 部分阅读有关引导加载器的更多信息。
:::

::::

2. 配置 Nginx 以使用 PHP-FPM。

以下是如何设置 Nginx 以与 PHP-FPM 一起工作的示例：

```nginx /etc/nginx/sites-enabled/spiral.conf
server {
    listen 80;
    listen [::]:80;
    server_name example.com;
    root /srv/example.com/public;
 
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
 
    index app.php;
 
    charset utf-8;
 
    location / {
        try_files $uri $uri/ /app.php?$query_string;
    }
 
    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }
 
    error_page 404 /app.php;
 
    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.1-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
 
    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

## Deployer

使用 [Deployer](https://deployer.org/) 等自动化工具可以自动执行部署过程并处理数据库迁移、缓存清除等任务。它们还允许更快的部署和回滚过程，并且可以与负载均衡器和监控系统等其他工具集成。这可以使部署过程更高效，更不容易出错，并且更容易跟踪当前生产服务器上的代码版本，从而在发生错误时难以回滚。

Deployer 是一种流行的部署工具，可用于自动化 Spiral 应用程序的部署过程。使用 Deployer 的优点之一是它可以提供“零停机”部署。

零停机部署是一种允许你在不中断对用户的服务的情况下更新应用程序的技术。这是通过在旧版本旁边部署应用程序的新版本，然后在新版本准备就绪后将其交换来实现的。这允许无缝过渡，因为用户不会受到部署过程中任何停机时间的影响。

Deployer 通过在不同的目录中创建应用程序的新版本并切换到新版本的符号链接来提供零停机部署的功能。

### 安装

要在你的项目中安装 Deployer，请运行以下命令：

```terminal
composer require --dev deployer/deployer
```

### 配置

要在你的项目中初始化 Deployer，请运行：

```terminal
./vendor/bin/dep init
```

Deployer 将向你提出几个问题，完成后你将拥有一个 `deploy.php` 或 `deploy.yaml` 文件。这是你的部署配方。它包含主机、任务，并且需要其他配方。

这是一个 `deploy.php` 文件的示例：

```php deploy.php
namespace Deployer;

require 'recipe/spiral.php';

set('repository', 'https://github.com/xxx/my-app');

add('shared_files', []);
add('shared_dirs', []);
add('writable_dirs', []);

host('example.org')
    ->set('remote_user', 'deployer')
    ->set('deploy_path', '/var/www/my-app');

after('deploy:failed', 'deploy:unlock');

desc('Deploys your project');
task('deploy', [
    'deploy:prepare',
    'deploy:environment',
    'deploy:vendors',
    'spiral:encrypt-key',
    'spiral:configure',
    'deploy:download-rr',
    'deploy:publish',
    'deploy:restart-rr'
]);
```

要连接到远程主机，我们需要指定一个标识密钥或私钥。我们可以将我们的标识密钥直接添加到主机定义中，但最好将其放在 `~/.ssh/config` 文件中：

```bash ~/.ssh/config
Host example.org
  IdentityFile ~/.ssh/id_rsa
```

现在让我们配置我们的服务器。由于我们的主机没有用户部署程序，我们将通过 `-o remote_user=root` 重写 `remote_user` 以进行配置。

```terminal
dep provision -o remote_user=root
```

Deployer 将在配置期间向你提出几个问题：php 版本、数据库类型等。接下来，Deployer 将配置我们的服务器并创建 deployer 用户。配置大约需要 5 分钟，并将安装运行网站所需的一切。一个新网站将被配置在 `deploy_path`。

### 部署

要部署你的应用程序，请运行以下命令：

```terminal
./vendor/bin/dep deploy
```

> **注意**
> 如果部署失败，Deployer 将打印一条错误消息以及哪个命令未成功。很可能我们需要在 `.env` 文件或类似文件中配置正确的数据库凭据。

#### CI/CD

Deployer 可用于 CI/CD 管道。

> **查看更多**
> 在 [Deployer CI/CD](https://deployer.org/docs/7.x/ci-cd) 部分阅读更多相关信息。

<hr>

## Docker

Docker 允许轻松部署、扩展和管理应用程序容器。它还使得在本地复制生产环境和测试应用程序变得容易，这使得查找和修复问题更容易。

此方法涉及构建应用程序的 Docker 镜像，然后在生产服务器上运行该镜像。

### Dockerfile

这是一个 `Dockerfile` 的示例，可用于构建 Spiral 应用程序的 Docker 镜像：

```dockerfile 
# 此示例将使用应用程序根目录作为 docker 上下文
FROM php:8.2-cli-alpine3.17 as backend

RUN  --mount=type=bind,from=mlocati/php-extension-installer:1.5,source=/usr/bin/install-php-extensions,target=/usr/local/bin/install-php-extensions \
      install-php-extensions opcache zip xsl dom exif intl pcntl bcmath sockets && \
     apk del --no-cache  ${PHPIZE_DEPS} ${BUILD_DEPENDS}

WORKDIR /app

ENV COMPOSER_ALLOW_SUPERUSER=1
COPY --from=composer:2.3 /usr/bin/composer /usr/bin/composer
COPY ./composer.* .
RUN composer config --no-plugins allow-plugins.spiral/composer-publish-plugin false && \
    composer install --optimize-autoloader --no-dev

COPY --from=spiralscout/roadrunner:latest /usr/bin/rr /app

EXPOSE 8080/tcp

COPY ./ .

CMD ./rr serve -c .rr.yaml
```

你可以通过运行以下命令来构建镜像：

```terminal
docker build . -t my-application:latest
```

构建镜像后，你需要将其推送到 Docker 注册表或中心。

```terminal
docker push my-application:latest
```

你也可以使用版本号标记该镜像，例如 `my-application:1.0`。

```terminal
docker build . -t my-application:latest -t my-application:1.0
docker push my-application:latest
docker push my-application:1.0
```

### Docker Compose

使用 Docker 的优点之一是，你可以通过 `docker-compose` 文件和外部 `.env` 文件管理应用程序的环境变量，而不必在容器内部创建 `.env` 文件。

这是一个 `docker-compose.yml` 文件的示例，可用于启动应用程序：

```yaml docker-compose.yaml
version: '3'
services:
  app:
    image: my-application:1.0
    ports:
      - "8080:8080"
    environment:
      - DEBUG=false
      - APP_ENV=production
      - ...
...
```

### 启动和停止应用程序

你可以通过运行以下命令来启动应用程序：

```terminal
docker-compose up -d
```

并通过运行以下命令停止它：

```terminal
docker-compose down
```

<hr>

## 源代码控制

部署应用程序的一种常用方法是使用源代码控制。这涉及将你的生产服务器连接到远程存储库，例如 Git 或 SVN，然后通过从存储库中提取最新的更改来部署应用程序。

### 示例

1. 通过 SSH 连接到生产服务器
2. 导航到服务器上应用程序的根目录
3. 运行命令 `git pull origin master` 以从远程存储库中提取最新的更改
4. 提取更改后，运行 `composer install --optimize-autoloader --no-dev` 以安装依赖项
5. 运行数据库迁移和其他必要的控制台命令
6. 重新启动 RoadRunner 服务器工作进程 `./rr reset`

> **警告：**
> 在没有自动化的情况下使用此策略会使部署过程更加耗时并且更容易出错。每次需要部署时，开发人员都需要手动从存储库中提取代码并按特定顺序运行任何必要的控制台命令。这增加了人为错误的风险，例如以错误的顺序运行命令或忘记运行命令，这可能会导致应用程序出现问题。

<hr>

## 文件传输

部署 Spiral 应用程序的最简单方法之一是通过基本的文件传输。这可以使用 FTP 或 SCP 来完成。该过程涉及将应用程序文件从你的本地计算机上传到生产服务器。

### 示例

1. 使用 FTP 客户端（FileZilla、WinSCP）连接到生产服务器
2. 导航到服务器上应用程序的根目录
3. 选择本地计算机上的所有应用程序文件并将它们上传到服务器
4. 上传完成后，运行 `composer install --optimize-autoloader --no-dev` 以安装依赖项
5. 运行数据库迁移和其他必要的控制台命令
6. 重新启动 RoadRunner 服务器工作进程 `./rr reset`

> **警告：**
> 这种策略被认为不如其他方法高效且安全。手动上传所有文件可能需要很长时间，并且如果传输中断，存在数据丢失或损坏的风险。此外，很难跟踪当前生产服务器上的代码版本，这使得在发生错误时难以回滚。
