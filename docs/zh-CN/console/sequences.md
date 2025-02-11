# Console — 序列

Spiral 提供了一种将一系列控制台命令或闭包按特定顺序分组并执行的方式。这些命令和闭包的组被称为“序列”，并且有两种类型：`configure` (配置) 序列和 `update` (更新) 序列。

使用控制台序列可以方便地自动化应用程序中的常见任务或操作，并有助于确保它们以一致且正确的顺序执行。

## 命令注册

### 配置序列

配置序列旨在用于与设置或配置应用程序相关的任务。这些命令在调用 `php app.php configure` 命令后运行。

以下是配置序列的示例：

```php
use Symfony\Component\Console\Output\OutputInterface;
use Psr\Container\ContainerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ConsoleBootloader $console): void
    {
        $console->addConfigureSequence(
            sequence: 'generate:keys', 
            header: '<info>Generating SSH keys for the application...</info>'
        );
        
        // 在序列中添加闭包
        // 它支持参数的自动注入
        $console->addConfigureSequence(
            static function(OutputInterface $output, ContainerInterface $container): void {
                // do something
            }, 
            '<info>Caching something...</info>'
        );
    }
}
```

### 更新序列

更新序列旨在用于与更新或修改应用程序相关的任务。这些命令在调用 `php app.php update` 命令后运行。

以下是更新序列的示例：

```php
use Symfony\Component\Console\Output\OutputInterface;
use Psr\Container\ContainerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ConsoleBootloader $console): void
    {
        $console->addUpdateSequence('db:migrate', '<info>Database migration...</info>');
        
        // 在序列中添加闭包
        // 它支持参数的自动注入
        $console->addConfigureSequence(
            static function(OutputInterface $output, ContainerInterface $container): void {
                // do something
            }, 
            '<info>Caching something...</info>'
        );
    }
}
```

## 自定义序列

除了这些预定义的序列之外，Spiral 还允许您创建自定义序列。自定义控制台序列是用户创建和管理的，一组按特定顺序运行的控制台命令或闭包。

以下是更新序列的示例：

```php app/src/Application/Bootloader/AppBootloader.php
use Symfony\Component\Console\Output\OutputInterface;
use Psr\Container\ContainerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ConsoleBootloader $console): void
    {
        $console->addSequence(
            name: 'cache_everything', 
            sequence: 'route:cache',
            header: '<info>Route caching...</info>',
            footer: '<info>Route caching completed.</info>'
        );
         
        $console->addSequence(
            name: 'cache_everything', 
            sequence: 'config:cache',
            header: '<info>Config caching...</info>',
            footer: '<info>Config caching completed.</info>'
        );
        
        // ... 
    }
}
```

自定义序列可以从扩展 `Spiral\Console\Sequence\SequenceCommand` 类的控制台命令运行。

以下是自定义序列命令的示例：

```php
namespace App\Command;

use Psr\Container\ContainerInterface;
use Spiral\Console\Config\ConsoleConfig;

final class CacheEverythingCommand extends SequenceCommand
{
    protected const NAME = 'cache:everything';
    protected const DESCRIPTION = 'Cache everything in the project';

    public function perform(ConsoleConfig $config, ContainerInterface $container): int
    {
        $this->info('Caching everything in the project...');
        $this->newLine();

        return $this->runSequence($config->getSequence('cache_everything'), $container);
    }
}
```

自定义序列可以是一个有用的工具，用于自动化应用程序中的任务或操作，并有助于简化您的开发和维护流程。

## 序列执行

将命令和闭包添加到序列后，可以使用 `php app.php configure` 命令执行配置序列，或者使用 `php app.php update` 命令执行更新序列。控制台将显示每个命令或闭包在运行时的描述，以及命令或闭包生成的任何输出。
