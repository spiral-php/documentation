# 控制台 — 入门

控制台组件让你在应用中轻松创建和管理控制台命令。通过利用 `symfony/console` 包的功能，Console 组件提供了一个便捷的界面来处理控制台命令。

所有提供的应用骨架都默认包含 Console 组件。要在替代构建中启用该组件，请确保要求 composer 包 `spiral/console` 并修改应用启动加载器：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\CommandBootloader::class,
        // ...
    ];
}
```

阅读更多关于启动加载器的信息，请参阅 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\CommandBootloader::class,
    // ...
];
```

阅读更多关于启动加载器的信息，请参阅 [Framework — Bootloaders](../framework/bootloaders.md) 章节。
:::

::::

要调用应用命令，只需运行：

```terminal
php app.php command:name
```

获取可用命令列表：

```terminal
php app.php list
```

获取关于特定命令的帮助：

```terminal
php app.php help command:name
```

## 在应用中调用

可以在你的应用或应用测试中调用控制台命令。这对于创建测试的模拟数据、自动预配置数据库或执行需要使用控制台命令的其他任务非常有用。

要从你的应用或测试中调用控制台命令，你可以使用 `Spiral\Console\Console` 服务。该服务提供了一个 `run()` 方法，允许你通过其名称和参数执行控制台命令。

以下是一个例子，展示了如何在你的应用中使用它来调用控制台命令：

```php
use Spiral\Console\Console;
use Symfony\Component\Console\Input\ArrayInput;
use Symfony\Component\Console\Output\BufferedOutput;

// ...

public function test(Console $console): string
{
    $input = new ArrayInput([
        '--mount' => '.env',
        '-p' => '{encrypt-key}'
    ]);
    
    $output = new BufferedOutput();
    
    return $console->run('encrypt:key', $input, $output)->fetch();
}
```

## Symfony/Console

Spiral Console 分发器基于强大的 [Symfony Console](http://symfony.com/doc/current/components/console/introduction.html) 组件构建。

> **注意**
> 你可以在你的 CLI 应用中注册原生的 Symfony 命令。

## 配置

要将自定义配置应用于 Console 组件，请使用 `Spiral\Config\ConfiguratorInterface` 或在 `app/config/console.php` 中创建一个配置文件：

```php
return [
     // 应用名称
     'name'      => null,
     
     // 应用版本
     'version'   => null,
     
     // 应用命令列表（如果禁用自动发现）
     'commands'  => [],
     
     // 在 `app configure` 中运行的命令和序列列表
     'configure' => [],
     
     // 在 `app update` 中运行的命令和序列列表
     'update'    => []
];
```

你可以在应用启动期间通过 `Spiral\Bootloader\ConsoleBootloader` 修改其中一些值。要注册一个新的用户命令：

```php
public function boot(ConsoleBootloader $console): void
{
    $console->addCommand(MyCommand::class);
}
```

> **注意**
> 默认情况下，该组件配置为自动发现位于 `app/src` 目录中的命令。

## 与 RoadRunner 的连接

**请注意，控制台命令是在 RoadRunner 服务器外部调用的。如果你的任何命令必须与其通信，请确保运行应用服务器的实例。**
