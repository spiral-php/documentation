# 框架 — 配置对象

Spiral 使用配置对象来分离应用程序的启动和运行时阶段，并提供一个易于访问的配置源。

使用配置对象的主要好处是，它们允许在启动过程中轻松修改配置，同时确保配置在运行时保持不可变。这有助于防止对配置的意外更改，并使应用程序的行为更可预测。

![Application Control Phases](https://user-images.githubusercontent.com/67324318/186413037-e60f89fd-9313-44c5-b4f8-eb2585a77230.png)

**以下是使用配置对象的一些好处：**

- 配置对象允许在启动过程中轻松修改配置值，使得在不修改代码的情况下更改应用程序行为变得简单。
- 一旦应用程序启动，配置对象会冻结这些值，防止在运行时对配置进行意外更改。这有助于确保应用程序按预期运行。
- 配置对象集中了配置信息，使其易于定位和按需修改。这提高了代码库的整体组织性和可维护性。
- 通过将配置信息与应用程序的代码分开，配置对象有助于改进关注点分离，并使代码库更易于理解和维护。

## 配置提供程序

Spiral 提供了一个 `Spiral\Config\ConfiguratorInterface` 接口，允许轻松访问存储在配置文件中的配置值。它提供了检索和检查配置值是否存在的方法。

为了演示这一点，我们可以创建一个配置文件 `app/config/github.php`：

```php app/config/github.php
<?php

return [
    'access_token' => 'xxx-xxxx',
    // ...
];
```

你可以使用任何你想要的名字，但建议使用将使用此配置的服务名称。

> **注意**
> 你也可以使用 `json` 格式，或者扩展 `ConfiguratorInterface` 来添加自定义的配置读取器。

### 使用

要在你的服务中访问配置值，你可以按以下方式使用 `ConfiguratorInterface`：

```php
use Spiral\Config\ConfiguratorInterface;

final class GithubClient
{
    private readonly string $accessToken;
    
    public function __construct(ConfiguratorInterface $configurator)
    {
        if (!$configurator->exists('github')) {
            throw new \RuntimeException('Github configuration is missing');
        }
        
        $config = $configurator->get('github');
        $this->accessToken = $config['access_token'] ?? throw new \RuntimeException('Missing access token');
    }

    // ...
}
```

## 配置对象

以数组的形式读取配置不太方便。框架提供了 OOP 抽象来读取你的值。我们可以手动创建这个类，也可以通过 `spiral/scaffolder` 自动生成它：

```terminal
php app.php create:config github -r
```

> **注意**
> `-r` 选项用于从配置文件 `app/config/github.php` 中逆向工程配置结构。

生成的配置类位于 `app/src/Config/GithubConfig.php` 中：

```php app/src/Config/GithubConfig.php
namespace App\Config;

use Spiral\Core\InjectableConfig;

class GithubConfig extends InjectableConfig
{
    public const CONFIG = 'github';

    protected array $config = [
        'access_token' => '',
        // ...
    ];

    public function getAccessToken(): string
    {
        return $this->config['access_token'];
    }
}
```

基类 `Spiral\Core\InjectableConfig` 允许你在代码中直接请求此对象，而无需任何 IoC 容器配置。常量 `CONFIG` 包含配置文件的名称。

> **注意**
> 也可以根据需要自定义生成的类并添加额外的方法。

```php
use App\Config\GithubConfig;

final class GithubClient
{
    private readonly string $accessToken;
    
    public function __construct(GithubConfig $config)
    {
        $this->accessToken = $config->getAccessToken();
    }

    // ...
}
```

> **注意**
> 配置对象提供了只读 API，在运行时更改值是不可能的，以防止在长时间运行的应用程序中出现不希望的副作用。

每个 Spiral 组件都提供你可以在你的应用程序中使用的配置对象。

通过使用配置类，可以轻松访问和管理 Spiral 应用程序中的配置值，同时享受使用 OOP 抽象和改进代码组织的优势。

## Bootloader 中的默认配置

使用自定义的 bootloader 来定义默认的配置值，以避免创建不必要的文件。环境变量可以用作默认值。

这在以下情况下非常有用：大多数应用程序的默认配置就足够了，并且可以避免创建不必要的配置文件。

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Config\ConfiguratorInterface;

final class GithubBootloader extends Bootloader
{
    public function init(ConfiguratorInterface $configurator, EnvironmentInterface $env): void
    {
        $configurator->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN')
            'authentication_type' => $env->get('GITHUB_AUTHENTICATION_TYPE', 'token')
        ]);
    }
}
```

这种方法对于设置所有环境通用的默认配置值，或者在未设置特定环境变量时提供后备值非常有用。

如果你想使用默认配置，你可以简单地删除 `app/config/github.php` 文件。

> **注意**
> 此功能允许设置可以轻松覆盖的默认配置值，同时在没有配置文件时提供后备。

## 自动配置

一些组件将公开自动配置 API，以便在应用程序启动期间更改其设置。通常，这种 API 可以通过组件的 bootloader 使用。

> **注意**
> 例如 `HttpBootloader`->`addMiddleware`。

我们可以在我们的 Bootloader 中提供我们的自动配置 API。为此目的使用 `ConfiguratorInterface`->`modify`。我们的 Bootloader 将被声明为单例，以加快处理速度。

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Config\Patch\Set;
use Spiral\Core\Container\SingletonInterface;

class GithubBootloader extends Bootloader implements SingletonInterface
{
    public function __construct(
        private readonly ConfiguratorInterface $configurator
    ) {
    }

    public function init(): void
    {
        $configurator->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN')
            'authentication_type' => $env->get('GITHUB_AUTHENTICATION_TYPE', 'token')
        ]);
    }

    public function setAccessToken(string $token): void
    {
        $this->configurator->modify(
          GithubConfig::CONFIG, 
          new Set('access_token', $token)
        );
    }
}
```

现在，我们可以通过另一个 bootloader 中的严格 API 修改配置：

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

class SomeBootloader extends Bootloader
{
    public function init(GithubBootloader $github): void
    {
        $github->setAccessToken('xxx-xxxx');
    }
}
```

## 配置生命周期

该框架提供了一种安全机制，以确保在任何组件请求配置对象后，你不会更改配置值（也称为配置被冻结）。

为了演示这一点，将 `GithubConfig` 注入到 `SomeBootloader` 中：

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;

final class SomeBootloader extends Bootloader
{
    public function boot(GithubBootloader $github, GithubConfig $config): void
    {
        // forbidden
        $github->setAccessToken(800);
    }
}
```

你将收到这个异常 `Spiral\Config\Exception\ConfigDeliveredException`：*无法修补配置 `github`，配置对象已交付。*

> **警告**
> 建议在 `init` 方法中设置默认值和更改配置。并且只在 `boot` 方法中请求配置文件。在 `boot` 方法之前调用 `init` 方法。这将确保当 `boot` 方法被调用时，配置将处于正确的状态。
