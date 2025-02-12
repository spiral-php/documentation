# 入门指南 — 配置

Spiral 让您可以通过配置文件轻松设置应用程序。所有配置文件都位于 `app/config` 目录中，它们允许您配置诸如数据库连接、设置缓存存储、管理队列等内容。

您甚至不必自己创建任何配置文件，因为 Spiral 附带了默认设置。但如果您愿意，您也可以使用环境变量来更改应用程序的主要参数。

> **注意**
> 缓存配置文件不是必需的，因为它们仅在应用程序引导期间加载一次，除非重新启动应用程序，否则不会重新加载。

## 环境变量

使用环境变量是将应用程序的配置与代码本身分离的好方法。这使得存储敏感信息（如数据库凭据、API 密钥和其他您不想硬编码到应用程序中的配置）变得容易。

Spiral 通过 `Spiral\DotEnv\Bootloader\DotenvBootloader` 类与 [Dotenv](https://github.com/vlucas/phpdotenv) 集成。该引导程序负责从 `.env` 文件加载环境变量，并使它们对应用程序可用。

通常的做法是在一个新项目中包含一个 `.env.sample` 文件，该文件可以用作设置环境变量的指南。

<details>
  <summary>点击显示 .env.sample</summary>

```dotenv .env
# Environment (prod or local)
APP_ENV=local

# Debug mode set to TRUE disables view caching and enables higher verbosity
DEBUG=true
VERBOSITY_LEVEL=verbose # basic, verbose, debug

# Set to an application specific value, used to encrypt/decrypt cookies etc
ENCRYPTER_KEY=...

# Monolog
MONOLOG_DEFAULT_CHANNEL=default
MONOLOG_DEFAULT_LEVEL=DEBUG # DEBUG, INFO, NOTICE, WARNING, ERROR, CRITICAL, ALERT, EMERGENCY

# Queue
QUEUE_CONNECTION=roadrunner

# Cache
CACHE_STORAGE=roadrunner

# Telemetry
TELEMETRY_DRIVER=null

# Serializer
DEFAULT_SERIALIZER_FORMAT=json # csv, xml, yaml

# Session
SESSION_LIFETIME=86400
SESSION_COOKIE=sid

# Authorization
AUTH_TOKEN_TRANSPORT=cookie
AUTH_TOKEN_STORAGE=session

# Mailer
MAILER_DSN=
MAILER_FROM="My site <no-reply@site.com>"
```

</details>

> **警告**
> 在 `$_SERVER` 或 `$_ENV` 超全局变量中定义的变量将优先于在 `.env` 文件中定义的相同变量的值。

来自 `.env` 文件中的值将被复制到您的应用程序环境中，并且可以通过 `Spiral\Boot\EnvironmentInterface` 或 `env` 函数使用。

### 可用的变量

| 变量                    | 描述                                                                                             |
|-----------------------------|-------------------------------------------------------------------------------------------------|
| `APP_ENV`                   | 当前应用程序环境。(`prod`, `stage`, `testing`, `local`)。默认值：`local`                          |
| `DEBUG`                     | 调试模式。默认值：`false`                                                                       |
| `TOKENIZER_CACHE_TARGETS`   | Tokenizer 的缓存目标。（布尔值）默认值：`false`                                                  |
| `ENCRYPTER_KEY`             | 加密密钥。                                                                                        |
| `VERBOSITY_LEVEL`           | 详细程度。(`basic`, `verbose`, `debug`)。默认值：`verbose`                                        |
| `MONOLOG_DEFAULT_CHANNEL`   | `app/config/monolog.php` 中的默认通道名称。默认值：`default`                                       |
| `QUEUE_CONNECTION`          | `app/config/queue.php` 中的队列连接名称。默认值：`sync`                                           |
| `BROADCAST_CONNECTION`      | 广播连接。(`log`, `null`, `centrifugo`)。默认值：`null`                                           |
| `CACHE_STORAGE`             | 来自 `app/config/cache.php` 的缓存存储。                                                          |
| `TELEMETRY_DRIVER`          | 遥测驱动程序。(`null`, `log`, `otel`)。默认值：`null`                                              |
| `LOCALE`                    | 默认区域设置。默认值：`en`                                                                       |
| `DEFAULT_SERIALIZER_FORMAT` | 默认序列化程序格式。(`json`, `serializer`) 默认值：`json`                                          |
| `AUTH_TOKEN_TRANSPORT`      | 授权令牌传输。(`cookie`, `header`)。默认值：`cookie`                                               |
| `AUTH_TOKEN_STORAGE`        | 授权令牌存储。(`session`, `cycle`)。默认值：`session`                                             |
| `SESSION_LIFETIME`          | 会话生命周期。默认值：`86400`                                                                     |
| `SESSION_COOKIE`            | 会话 cookie 名称。默认值：`sid`                                                                  |
| `VIEW_CACHE`                | 视图缓存。默认值：`DEBUG !== true`                                                                |
| `MAILER_DSN`                | Mailer DSN。                                                                                       |
| `MAILER_FROM`               | Mailer 发件人。默认值：`Spiral <sendit@local.host>`                                               |
| `MAILER_QUEUE_CONNECTION`   | Mailer 队列连接。默认值：`QUEUE_CONNECTION` 变量中的一个值                                         |

### 访问环境变量

您可以使用 `Spiral\Boot\EnvironmentInterface` 访问环境变量

```php
use Spiral\Boot\EnvironmentInterface;

final class GithubClient
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {}
    
    public function getAccessToken(): ?string
    {
        return $this->env->get('GITHUB_ACCESS_TOKEN');
    }
}
```

或者通过简短的函数 `env()`

```php 
return [
    'access_token' => env('GITHUB_ACCESS_TOKEN'),
    // ...
];
```

### 预处理

请记住，`.env` 中的值将被预处理，将进行以下更改：

| 值        | PHP 值    |
|-----------|-----------|
| true      | true      |
| (true)    | true      |
| false     | false     |
| (false)   | false     |
| null      | null      |
| (null)    | null      |
| empty     | ''        |

> **注意**
> 字符串周围的引号将被自动剥离。

## 配置

虽然环境变量是配置应用程序某些选项的好方法，但可能存在您需要对配置进行更复杂或更细粒度更改的情况，这些更改无法仅通过环境变量实现或不切实际。在这种情况下，您可以直接修改要更改的特定组件的配置文件。

例如，您可以更改默认的 HTTP 标头：

```php app/config/http.php
return [
    'basePath'   => '/',
    'headers' => [
        'Server' => 'Spiral',
        'Content-Type' => 'text/html; charset=UTF-8'
    ],
    'middleware' => [],
];
```

### 访问配置值

### 配置对象

在 Spiral 中，所有配置对象都是可注入的，这使得在您的应用程序中使用它们变得容易。

```php
use Spiral\Http\Config\HttpConfig;

final class HttpClient 
{
    private readonly string $basePath;

    public function __construct(
        HttpConfig $config // <-- Container will automatically load values from app/config/http.php
    ) {
        $this->basePath = $this->config->getBasePath();
    }
}
```

Spiral 中的每个可注入配置类都包含一个 [CONFIG 常量](https://github.com/spiral/http/blob/master/src/Config/HttpConfig.php#L19)，它定义了相应配置文件的名称。当容器解析可注入的配置对象时，它会自动从配置文件加载所有值，并将它们分配给配置对象的 `$config` 属性。

> **注意**
> 当配置对象加载其关联的配置文件时，它会自动将其与在配置类中定义的默认设置合并。这意味着您不必在配置文件中包含每个选项，而只需包含要更改的选项。如果配置文件中不存在某个值，则使用默认设置作为后备。

例如，对于 **HTTP 配置**：

```php spiral/framework/src/Http/src/Config/HttpConfig.php
final class HttpConfig extends InjectableConfig
{
    const CONFIG = 'http';
    
    // ...
}
```

> **注意**
> 请参阅相关文档部分，了解每个组件配置的参考。

### 确定应用程序环境

当前应用程序环境由 `APP_ENV` 变量确定。您可以使用 `Spiral\Boot\Environment\AppEnvironment` 可注入枚举类访问此值。

> **另请参阅**
> 在 [高级 — 容器注入器](../container/injectors.md#enum-injectors) 部分阅读有关可注入枚举的更多信息。

当您从容器中请求 `AppEnvironment` 时，它将自动注入一个包含正确值的枚举。

```php
use Spiral\Boot\Environment\AppEnvironment;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly AppEnvironment $env
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->env->isProduction()) {
                // ...
            }
            
            // ...
        }
    }
}
```

### 确定调试模式

当前调试模式由 `DEBUG` 变量确定。您可以使用 `Spiral\Boot\Environment\DebugMode` 可注入枚举类访问此值。

```php
use Spiral\Boot\Environment\DebugMode;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly DebugMode $debug
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->debug->isEnabled()) {
                // ...
            }
            
            // ...
        }
    }
}
```

### 确定详细程度

当前详细程度由 `VERBOSITY_LEVEL` 变量确定。您可以使用 `Spiral\Exceptions\Verbosity` 可注入枚举类访问此值。

```php
use Spiral\Exceptions\Verbosity;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly Verbosity $verbosity
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->verbosity === Verbosity::BASIC)
                // ...
            }
            
            if ($this->verbosity === Verbosity::VERBOSE)
                // ...
            }
            
            // ...
        }
    }
}
```

<hr>

## 接下来做什么？

现在，通过阅读一些文章深入了解基础知识：

* [配置对象](../framework/config.md)
* [高级 — 容器注入器](../container/injectors.md)
* [内核和环境](../framework/kernel.md)
