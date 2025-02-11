# 高级特性 — 原子锁

Spiral 完整集成了 [RoadRunner locks](https://roadrunner.dev/docs/plugins-locks) 插件，这使你
能够在你的应用程序中管理资源锁。通过这个组件，你可以轻松管理应用程序的关键部分并防止竞争条件、数据损坏以及可能在
多进程环境中发生的其他同步问题。

为了启用与 RoadRunner 的集成，Spiral 通过
[spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) 包提供了内置支持。

> **警告**
> 锁仅在 `spiral/roadrunner-bridge` 包的 `3.2` 及更高版本中可用。

## 安装

首先，你需要安装 [Roadrunner bridge](../start/server.md#roadrunner-bridge) 包。安装完成后，
将 `Spiral\RoadRunnerBridge\Bootloader\LockBootloader` 添加到你的 Kernel 类的引导程序列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\LockBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读关于引导程序的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\LockBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读关于引导程序的更多信息。
:::

::::

就这么简单！RoadRunner 端不需要额外的配置。

## 使用

以下是一个关于如何在你的应用程序中使用锁的简短示例：

```php
$locks = $container->get(\RoadRunner\Lock\LockInterface::class);
$id = $lock->lock('pdf:create');

// Your logic for creating a PDF file

$lock->release('pdf:create', $id);
```

> **警告**
> RoadRunner 锁插件此刻使用内存存储来存储关于锁的信息。当使用多个 RoadRunner 实例时，每个实例都将拥有自己用于锁的内存存储。因此，如果一个进程在一个 RoadRunner 实例上获取了锁，它将不会知道其他实例上的锁状态。

### 获取锁

锁定资源确保只有一个进程可以一次访问它，从而防止数据冲突并确保一致性。

#### 基本的锁获取

在指定的资源上获取一个独占锁。

```php
$id = $lock->lock('pdf:create');
```

这是获取锁的最简单形式。它尝试锁定由 `pdf:create` 标识的资源。如果成功，它将返回该锁的标识符。

#### 带生存时间 (TTL) 的锁

```php
$id = $lock->lock('pdf:create', ttl: 10);
// or
$id = $lock->lock('pdf:create', ttl: new \DateInterval('PT10S'));
```

这些行演示了如何使用 TTL（生存时间）来获取锁。第一行设置了 10 微秒的 TTL，而第二行使用一个 DateInterval 对象来设置 10 秒的 TTL。TTL 用于指定锁应该保持多长时间，然后自动释放。

#### 带等待时间的锁

```php
$id = $lock->lock('pdf:create', wait: 5);
// or
$id = $lock->lock('pdf:create', wait: new \DateInterval('PT5S'));
```

这些行展示了如何使用指定的等待时间来获取锁。如果锁无法立即使用，进程将等待给定的时间（第一行是 5 微秒，第二行是 5 秒），然后放弃。

#### 带 ID 的锁

```php
$id = $lock->lock('pdf:create', id: '14e1b600-9e97-11d8-9f32-f2801f1b9fd1');
```

这行演示了使用特定标识符获取锁。这对于跟踪或记录目的很有用，允许你为锁指定自定义标识符。

### 获取读锁

在并发编程中，获取读锁允许多个进程同时访问资源以进行读取，同时仍然阻止独占写访问。

有类似的方法可以获取读锁：

```php
$id = $lock->lockRead('pdf:create', ttl: 100000);
// or
$id = $lock->lockRead('pdf:create', ttl: new \DateInterval('PT10S'));

// Acquire lock and wait 5 microseconds until lock will be released
$id = $lock->lockRead('pdf:create', wait: 5);
// or
$id = $lock->lockRead('pdf:create', wait: new \DateInterval('PT5S'));

// Acquire lock with id - 14e1b600-9e97-11d8-9f32-f2801f1b9fd1
$id = $lock->lockRead('pdf:create', id: '14e1b600-9e97-11d8-9f32-f2801f1b9fd1');
```

### 释放锁

释放锁是处理并发控制的一个重要部分。它确保为其他进程释放资源以供使用。

> **警告**
> 一旦完成需要独占或共享访问资源的任务，就应该始终释放锁。这确保了其他进程可以在需要时获取锁。

要释放先前获取的独占或读锁：

```php
$id = $lock->lock('pdf:create');

// Your logic for creating a PDF file

$lock->release('pdf:create', $id);
```

#### 强制释放锁

在某些情况下，你可能需要释放锁，而不知道锁标识符。例如，如果一个进程在持有锁时崩溃，则锁将不会被释放。

```php
$lock->forceRelease('pdf:create');
```

### 检查锁

要检查资源上是否存在锁，请使用 `exists()` 方法：

```php
$status = $lock->exists('pdf:create');

if ($status) {
    // Lock exists
} else {
    // Lock not exists
}
```

### 更新 TTL

在某些情况下，你可能需要更新锁的 TTL。例如，如果一个进程正在执行长时间运行的任务，你可能希望延长 TTL 以防止在任务完成之前释放锁。

```php
// Add 10 microseconds to lock ttl
$lock->updateTTL('pdf:create', $id, 10);
// or
$lock->updateTTL('pdf:create', $id, new \DateInterval('PT10S'));
```

## Symfony 集成

我们还提供了一个包，它将锁驱动程序添加到
[Symfony lock](https://symfony.com/doc/current/components/lock.html) 组件。

要安装该包，请运行以下命令：

```terminal
composer require roadrunner-php/symfony-lock-driver
```

安装完成后，你需要创建一个引导程序类，该类将在容器中注册锁驱动程序：

```php app/src/Application/Bootloader/LockBootloader.php
<?php

declare(strict_types=1);

namespace App\Application\Bootloader;

use RoadRunner\Lock\LockInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\Environment\AppEnvironment;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Goridge\RPC\RPCInterface;
use Spiral\RoadRunner\Symfony\Lock\RoadRunnerStore;
use Spiral\RoadRunnerBridge\Bootloader\RoadRunnerBootloader;
use Symfony\Component\Lock\LockFactory;
use Symfony\Component\Lock\PersistingStoreInterface;
use Symfony\Component\Lock\Store\InMemoryStore;
use Symfony\Component\Lock\Store\RedisStore;

final class LockBootloader extends Bootloader
{
    public function defineDependencies(): array
    {
        return [
            \Spiral\RoadRunnerBridge\Bootloader\LockBootloader::class
        ];
    }

    public function defineSingletons(): array
    {
        return [
            LockFactory::class => [self::class, 'initLockFactory'],
        ];
    }

    protected function initLockFactory(LockInterface $rrLock, EnvironmentInterface $env): LockFactory
    {
        $driver = $env->get('LOCK_DRIVER', 'roadrunner');
        $defaultTtl = $env->get('LOCK_DRIVER_TTL', 100);

        $store = match ($driver) {
            'memory' => new InMemoryStore(), // for testing purposes
            'roadrunner' => new RoadRunnerStore($rrLock, initialWaitTtl: $defaultTtl),
            default => throw new \InvalidArgumentException("Unknown lock driver: {$driver}"),
        };

        return new LockFactory($store);
    }
}
```

现在你可以在你的应用程序中使用 Symfony 锁组件：

```php
use Symfony\Component\Lock\LockFactory;

$factory = $container->get(LockFactory::class);
$lock = $factory->createLock('pdf-creation');

if ($lock->acquire()) {
    // The resource "pdf-creation" is locked.
    // You can compute and generate the invoice safely here.

    $lock->release();
}
```

> **注意**
> 在 [Symfony documentation](https://symfony.com/doc/current/components/lock.html) 中阅读更多关于 Symfony 锁组件的信息。
