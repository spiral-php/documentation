# 测试 — 入门

Spiral 在设计时非常强调测试。它内置了对 [PHPUnit](https://phpunit.de/) 的支持，并且为您的应用程序预先配置了 `phpunit.xml` 文件。

Spiral 应用程序的默认目录结构包含一个 `tests` 目录，该目录包含两个子目录：`Feature` 和 `Unit`。

**Unit** 测试旨在测试代码的小型、隔离的部分，通常侧重于单个方法。这些测试不会启动整个 Spiral 应用程序，**因此无法访问数据库或其他框架服务**。

另一方面，**Feature** 测试旨在测试更大范围的代码，包括多个对象之间的交互，甚至是完整的 HTTP 请求到 JSON 端点。这些测试提供了对应用程序更全面的覆盖，并让您更有信心它按预期运行。

Spiral 提供了 `spiral/testing` 软件包，以帮助开发人员进行应用程序测试。该软件包提供了各种辅助方法，可以简化编写 Spiral 应用程序测试的过程。

## 配置

为了运行测试，您可能需要设置某些环境变量。一种方法是使用 `phpunit.xml` 文件。PHPUnit 测试框架使用此文件来配置测试环境。

**以下是一个示例：**

```xml phpunit.xml

<phpunit>
    // ...
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="QUEUE_CONNECTION" value="sync"/>
        <env name="MONOLOG_DEFAULT_CHANNEL" value="stderr"/>
        <env name="CACHE_STORAGE" value="array"/>
        <env name="TOKENIZER_CACHE_TARGETS" value="true" />
        <env name="CYCLE_SCHEMA_CACHE" value="true" />
        // ...
    </php>
</phpunit>
```

> **警告**
> 当您在 Docker 容器中运行测试时，Docker 中的设置比 `phpunit.xml` 中的设置更重要。如果它们不匹配，这可能会导致问题。请确保 Docker 设置适合您的测试。

> **注意**
> 强烈建议启用 `TOKENIZER_CACHE_TARGETS` 和 `CYCLE_SCHEMA_CACHE` 以增强测试性能。通过这样做，您可以缓存 tokenizer 和 ORM schema，这意味着它们不会在每次测试迭代中执行。但是，请注意，每当您更改代码或实体的 schema 时，务必清除此缓存，以确保测试使用最新的配置运行。

## 单元测试

另一方面，单元测试侧重于代码的小型、隔离的部分，应该扩展 `PHPUnit\Framework\TestCase` 类。单元测试不会启动整个 Spiral 应用程序，因此它们无法访问数据库或其他框架服务。

**以下是一个简单的单元测试示例：**

```php tests/Unit/UserTest.php
namespace Tests\Unit;

use PHPUnit\Framework\TestCase;

final class UserTest extends TestCase
{
    public function testGetId(): void
    {
        $user = new User(id: 101);
        $this->assertSame(101, $user->getId());
    }
}
```

## 功能测试

功能测试测试代码的更大一部分，应该扩展 `Tests\TestCase` 抽象类。这个类是专门设计用来引导您的应用程序，模拟 Web 服务器的行为，但使用测试环境变量。

**以下是一个简单的功能测试示例：**

```php tests/Feature/Controller/UserController/ShowActionTest.php
namespace Tests\Feature\Controller\UserController;

use Tests\TestCase;

final class ShowActionTest extends TestCase
{
    public function testShowPageNotFoundIfUserNotExist(): void
    {
        $http = $this->fakeHttp();
        $response = $http->get('/user/1');
        $response->assertNotFound();
    }
}
```

### 环境变量

`Tests\TestCase` 类包含一个功能，允许您为测试用例设置环境变量。如果您需要根据环境变量测试特定行为，这可能会很有用。

```php
use Tests\TestCase;

final class SomeTest extends TestCase
{
    public const ENV = [
        'DEBUG' => false,
        // ...
    ];

    public function testSomeFeature(): void
    {
        //
    }
}
```

您也可以使用 PHP 属性定义 ENV 变量。这允许更精细地控制测试环境。

```php
use Tests\TestCase;
use Spiral\Testing\Attribute\Env;

final class SomeTest extends TestCase
{
    #[Env('DEBUG', false)]
    #[Env('APP_ENV', 'production')]
    public function testSomeFeature(): void
    {
        //
    }
}
```

----

### 与容器交互

`Tests\TestCase` 类还包含一个功能，允许您与容器交互。

#### 获取容器实例

```php
$container = $this->getContainer();
````

#### 检查服务是否未在容器中注册

```php
$this->assertContainerMissed(\Spiral\Queue\QueueConnectionProviderInterface::class);
```

#### 检查容器是否可以使用自动装配创建对象

```php
$this->assertContainerInstantiable(\Spiral\Queue\QueueConnectionProviderInterface::class);
```

#### 检查容器是否具有给定的别名并已绑定

```php
$this->assertContainerBound(\Spiral\Queue\QueueConnectionProviderInterface::class);
```

检查容器是否绑定到特定的类

```php
$this->assertContainerBound(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class
);
```

检查容器是否绑定到具有参数的特定类

```php
$this->assertContainerBound(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class,
    ['foo' => 'bar']
);
```

您还可以使用回调函数进行额外的检查

```php
$this->assertContainerBound(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class,
    ['foo' => 'bar'],
    function(\Spiral\Queue\QueueManager $manager) {
        $this->assertEquals(..., $manager->....)
    }
);
```

#### 检查容器是否绑定为单例

```php
$this->assertContainerBoundAsSingleton(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class
);
```

#### 检查容器是否未绑定为单例

```php
$this->assertContainerBoundNotAsSingleton(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class
);
```

#### 模拟容器中的服务

在某些情况下，您需要在容器中模拟服务。例如，身份验证服务。

```php
namespace Tests\Feature\Controller\UserController;

use Tests\TestCase;

final class ProfileActionTest extends TestCase
{
    public function testShowPageNotFoundIfUserNotExist(): void
    {   
        $auth = $this->mockContainer(\Spiral\Auth\ActorProviderInterface::class);
        $auth->shouldReceive('getActor')->with(...)->once()->andReturnNull();

        $http = $this->fakeHttp();
        $response = $http->get('/user/profile');

        $response->assertNotFound();
    }
}
```

----

### 与配置交互

让我们假设我们有以下配置：

```php app/config/http.php
return [
    'basePath'   => '/',
    'headers'    => [
        'Content-Type' => 'text/html; charset=UTF-8',
    ],
    'middleware' => [],
];
```

#### 定义配置值

在某些情况下，您需要为特定的测试定义配置值。您可以使用 PHP 属性来完成。

```php
use Tests\TestCase;
use Spiral\Testing\Attribute\Config;

final class SomeTest extends TestCase
{
    #[Config('http.basePath', '/custom')]
    #[Config('http.headers.Content-Type', 'text/plain')]
    public function testSomeFeature(): void
    {
        //
    }
}
````

#### 检查配置是否与给定值匹配

```php
$this->assertConfigMatches('http', [
    'basePath'   => '/',
    'headers'    => [
        'Content-Type' => 'text/html; charset=UTF-8',
    ],
    'middleware' => [],
]);
```

#### 检查配置是否包含给定的片段

```php
$this->assertConfigHasFragments('http', [
    'basePath' => '/'
])
```

#### 获取配置

```php
/** @var array $config */
$config = $this->getConfig('http');
```

----

### 与目录交互

#### 检查目录别名是否存在

```php
$this-assertDirectoryAliasDefined('root');
```

#### 检查目录别名是否与给定值匹配

```php
$this->assertDirectoryAliasMatches('runtime', __DIR__.'src/runtime');
```

#### 清理目录

```php
$this->cleanupDirectories(
    __DIR__.'src/runtime/cache',
    __DIR__.'src/runtime/tmp'
);
```

#### 通过别名清理目录

```php
$this->cleanupDirectoriesByAliases(
    'runtime', 'cache', 'logs'
);
```

您也可以清理运行时目录

```php
$this->cleanUpRuntimeDirectory();
```

----

### 与控制台交互

#### 检查是否注册了命令

```php
$this->assertCommandRegistered('ping');
```

#### 运行控制台命令

您可以在测试用例中运行控制台命令并检查结果。

```php
$output = $this->runCommand('ping', ['site' => 'https://google.com']);
$this->assertStringContainsString('Pong', $output);
```

您也可以使用以下方法检查输出中的字符串

```php
$this->assertConsoleCommandOutputContainsStrings(
    'ping',
    ['site' => 'https://google.com'],
    ['Site found', 'Starting ping ...', 'Success!']
);
```

----

### 与 Bootloader 交互

`Tests\TestCase` 类还包含一个功能，允许您与启动加载器交互。

#### 检查是否注册了启动加载器

```php
$this->assertBootloaderLoaded(\MyPackage\Bootloaders\PackageBootloader::class);
```

#### 检查启动加载器是否未注册

```php
$this->assertBootloaderMissed(\MyPackage\Bootloaders\PackageBootloader::class);
```

----

### 与调度器交互

#### 检查是否注册了调度器

```php
$this->assertDispatcherRegistered(HttpDispatcher::class);
```

#### 检查调度器是否未注册

```php
$this->assertDispatcherMissed(HttpDispatcher::class);
```

#### 运行调度器

您可以通过传递一些绑定作为第二个参数来运行调度器。它将在具有绑定的范围内运行。

```php
$this->serveDispatcher(HttpDispatcher::class, [
    \Spiral\Boot\EnvironmentInterface::class => new \Spiral\Boot\Environment([
        'foo' => 'bar'
    ]),
]);
```

#### 检查调度器是否可以被服务

检查调度器是否可以在当前环境中被服务。

```php
$this->assertDispatcherCanBeServed(HttpDispatcher::class);
```

#### 检查调度器是否不能被服务

检查调度器是否不能在当前环境中被服务。

```php
$this->assertDispatcherCannotBeServed(HttpDispatcher::class);
```

#### 获取已注册的调度器

```php
/** @var class-string<\Spiral\Boot\DispatcherInterface>[] $dispatchers */
$dispatchers = $this->getRegisteredDispatchers();
```

----

### 与 Scaffolder 交互

我们可以提供测试脚手架命令的能力。

#### 断言生成的代码与预期相同

```php
$this->assertScaffolderCommandSame(
    'create:command',
    [
        'name' => 'TestCommand',
    ],
    expected: <<<'PHP'
    <?php
    
    declare(strict_types=1);
    
    namespace Spiral\Testing\Command;
    
    use Spiral\Console\Attribute\Argument;
    use Spiral\Console\Attribute\AsCommand;
    use Spiral\Console\Attribute\Option;
    use Spiral\Console\Attribute\Question;
    use Spiral\Console\Command;
    
    #[AsCommand(name: 'test:command')]
    final class TestCommand extends Command
    {
        public function __invoke(): int
        {
            // Put your command logic here
            $this->info('Command logic is not implemented yet');
    
            return self::SUCCESS;
        }
    }
    
    PHP,
    expectedFilename: 'app/src/Command/TestCommand.php',
    expectedOutputStrings: [
        "Declaration of 'TestCommand' has been successfully written into 'app/src/Command/TestCommand.php",
    ],
);
```

#### 断言生成的代码包含给定的字符串

```php
$this->assertScaffolderCommandContains(
    'create:command',
    [
        'name' => 'TestCommand',
        '--namespace' => 'App\Command',
    ],
    expectedStrings: [
        'namespace App\Command;',
    ],
    expectedFilename: 'app/src/TestCommand.php',
);
```

#### 断言抛出异常

```php
$this->expectException(RuntimeException::class);
$this->expectExceptionMessage('Not enough arguments (missing: "name").');

$this->assertScaffolderCommandSame(
    'create:command',
    [],
    '',
);
```

#### 断言运行命令并附带附加选项

```php
$this->assertScaffolderCommandContains(
    'create:command',
    [
        'name' => 'TestCommand',
        '-o' => 'foo',
    ],
    expectedStrings: [
        "#[Option(description: 'Argument description')]",
        'private bool $foo;'
    ],
);
```

----

## 运行测试

当您运行 `vendor/bin/phpunit` 命令时，它将自动在应用程序的 `tests` 目录中查找测试文件，并使用 `phpunit.xml` 文件中指定的配置运行它们。

**要运行 Spiral 应用程序中的测试，您可以简单地执行以下命令：**

```terminal
./vendor/bin/phpunit
```

尽情测试吧！

<hr>

## 接下来是什么？

现在，通过阅读一些文章深入了解基础知识：

* [队列和任务](../queue/configuration.md)
* [队列拦截器](../queue/interceptors.md)
