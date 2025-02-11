# 基础知识 — 缓存

缓存是一种通过将经常访问的数据存储在更快的存储介质（例如内存）中来显著提高应用程序性能的技术。这减少了从较慢的源（如数据库）重新计算或获取数据的需求。通过实现缓存层，应用程序可以快速从缓存中检索数据，而不是重新计算或从较慢的源中获取数据，这可以提高整体响应时间并降低较慢源的负载。

Spiral 中的 `spiral/cache` 组件提供了一种存储和检索数据的简化方法。该组件符合 [PSR-16 标准](https://www.php-fig.org/psr/psr-16/)，这使其成为熟悉 PHP 标准的开发人员的多功能选择。

## RoadRunner：缓存领域的革新者

当我们谈论 Spiral 的缓存能力时，RoadRunner 位于其核心。以下是关于 RoadRunner 你应该了解的信息：

- **用 Go 编写：** RoadRunner 注重速度和效率。Go 以其出色的性能指标而闻名，这使其成为高性能操作的自然选择。

- **专为高并发设计：** 它不仅仅是执行；它是并行执行。专为承担高性能任务而设计，它可以管理并发操作而不会出现任何问题。

- **通过 Key-Value 插件提供多样化的存储选择：** 借助 RoadRunner 的 [Key-Value 插件](https://roadrunner.dev/docs/plugins-kv/)，您有很多选择。选择像 Redis Server 和 Memcached 这样的知名产品，或者选择无服务器的方式，例如内存存储。

- **无需 PHP 扩展：** 使用 RoadRunner 的一个突出特点是消除了对 Redis 或 Memcache 的任何 PHP 扩展的需求。相反，您使用 RPC 与 RoadRunner 直接通信。

## 安装

要启用该组件，您只需要将 `Spiral\Cache\Bootloader\CacheBootloader` 添加到引导程序列表中：

> **警告**
> 确保您已安装 [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) 包。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Cache\Bootloader\CacheBootloader::class,
        \Spiral\RoadRunnerBridge\Bootloader\CacheBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Cache\Bootloader\CacheBootloader::class,
    \Spiral\RoadRunnerBridge\Bootloader\CacheBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::::

> **注意**
> `spiral/roadrunner-bridge` 包允许您使用 RoadRunner
> [kv 插件](https://roadrunner.dev/docs/plugins-kv/) 与 Spiral。此包为 KV 提供了 RPC API 以及用于应用程序的引导程序。

## 配置

要开始使用，请将您的配置文件放置在 `app/config/cache.php`。此文件是配置存储选项和设置默认存储的常用位置。

以下是您的 `cache.php` 配置可能的样子的一个示例：

```php app/config/cache.php
use Spiral\Cache\Storage\ArrayStorage;
use Spiral\Cache\Storage\FileStorage;

return [
    /**
     * -------------------------------------------------------------------------
     *  Default storage
     * -------------------------------------------------------------------------
     * 
     * The key of one of the registered cache storages to use by default.
     */
    'default' => env('CACHE_STORAGE', 'rr-redis'),
    
    /**
     * -------------------------------------------------------------------------
     *  Aliases
     * -------------------------------------------------------------------------
     * 
     * Aliases, if you want to use domain specific storages.
     */
    'aliases' => [
        'user-data' => 'rr-redis',
        'blog-data' => [
            'storage' => 'rr-redis',
            'prefix' => 'blog_'
        ],
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Storages
     * -------------------------------------------------------------------------
     * 
     * Here you may define all of the cache "storages" for your application as well as their types.
     */
    'storages' => [
        'rr-redis' => [
            'type' => 'roadrunner',
            'driver' => 'redis',
        ],

        'rr-local' => [
            'type' => 'roadrunner',
            'driver' => 'local',
        ],
        
        'local' => [
            'type' => 'array',
        ],
        
        'file' => [
            'type' => 'file',
            'path' => directory('runtime') . 'cache',
        ],
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Aliases for storage types
     * -------------------------------------------------------------------------
     */
    'typeAliases' => [
        'array' => ArrayStorage::class,
        'file' => FileStorage::class,
    ],
];
```

### 使用别名

别名是您引用缓存存储的快捷方式。它们简化了代码并使其更具可读性。

例如，您有一个名为 `rr-redis` 的存储，并且您希望在应用程序中将其引用为 `user-data`。`aliases` 部分可以帮助您做到这一点！

更重要的是，您可以使用 `Spiral\Cache\CacheStorageProviderInterface` 通过别名名称获取缓存存储。然后它会智能地映射到实际的存储，使您的代码更整洁、更易于管理。

### 缓存键前缀：

前缀在处理缓存别名时起着关键作用。它们确保您的缓存键保持唯一性，防止任何意外的重叠或冲突。通过将 `prefix` 附加到别名，该别名下的每个缓存键都会自动继承此前缀。这会产生一个结构化、有组织且无冲突的缓存系统。

将前缀合并到您的 cache.php 中很简单：

```php app/config/cache.php
return [
    // ...

    'aliases' => [
        'user-data' => [
            'storage' => 'rr-redis',
            'prefix' => 'user_'
        ],
    ],
];
```

在上面的示例中，`user-data` 别名下的任何缓存键都将自动添加 `user_` 前缀。因此，像 `profile` 这样的缓存键将被存储为 `user_profile`。

### RoadRunner KV 插件配置

以及 RoadRunner KV 插件的配置文件：

```yaml .rr.yaml
kv:
  local:
    driver: memory
    config:
      interval: 60
  redis:
    driver: redis
    config:
      addrs:
        - localhost:6379
...
```

> **了解更多**
> 在 [RoadRunner 文档](https://roadrunner.dev/docs/plugins-kv) 中阅读有关 RoadRunner KV 插件配置的更多信息。

## 用法

Spiral 的缓存组件允许您使用两个接口来使用缓存：`Psr\SimpleCache\CacheInterface` 和 `Spiral\Cache\CacheStorageProviderInterface`。

> **了解更多**
> 您可以使用 `cache` 和 `cacheManager` 原型属性。在 [基础知识 — 原型设计](../basics/prototype.md) 部分阅读有关原型的更多信息。

### 默认存储

通过使用 `Psr\SimpleCache\CacheInterface`，您可以访问在配置文件中定义的默认缓存存储。这可以使用依赖注入注入到类中，如下例所示：

```php
namespace App\Service;

use Psr\SimpleCache\CacheInterface;

final class UserService
{
    public function __construct(
        private readonly CacheInterface $cache,
    ) {
    }

    public function find(int $id): User
    {
        // ...
    }
}
```

### 缓存存储提供程序

另一方面，通过使用 `Spiral\Cache\CacheStorageProviderInterface`，您可以从配置文件中通过其字符串键获取特定的存储。这允许您访问特定域或用例的缓存存储。

```php
namespace App\Service;

use Spiral\Cache\CacheStorageProviderInterface;

class UserService
{
    private readonly CacheInterface $cache;
  
    public function __construct(CacheStorageProviderInterface $provider) 
    {
        $this->cache = $provider->storage('user-data');
    }

    public function find(int $id): User
    {
        //...
    }
}
```

在提供的示例中，特定于域的存储 `user-data` 被注入到 `UserService` 类中。这允许您访问专门为用户数据需求量身定制的缓存存储，并在整个类中使用它来从缓存中存储、检索和删除用户数据。

通过使用这种方法，您可以轻松地管理和维护应用程序不同部分的缓存数据，并根据需要轻松地在不同的存储选项之间切换。

### 检索项目

要从缓存中获取一个项目，您可以使用 `get` 方法。如果该项目不存在，它将返回 `null`。您还可以指定一个默认值，如果该项目不存在，则返回该值。

```php
$data = $this->cache->get('key');

$data = $this->cache->get('key', 'default');
```

如果您需要一次获取多个项目，您可以使用 `getMultiple` 方法并传入一个键数组。

```php
$data = $this->cache->getMultiple(['key', 'other']);
```

### 检查项目是否存在

如果您想检查缓存中是否存在某个项目，可以使用 `has` 方法。如果该项目存在，它将返回 `true`，如果不存在，则返回 `false`。

```php
if ($this->cache->has('key')) {
    // ...
}
```

### 存储项目

要在缓存中存储一个项目，您可以使用 `set` 方法。

```php
$this->cache->set(
    key: 'key', 
    value: ['some' => 'data'], 
    ttl: 3600
);
```

您还可以通过传入 `秒数` 或 `DateInterval` 或 `DateTimeInterface` 对象来指定该项目的存储时间（TTL）。

```php
$this->cache->set(
    key: 'key', 
    value: ['some' => 'data'], 
    ttl: \Carbon\Carbon::now()->addHour()
);
```

如果未指定值，则缓存值将无限期存储（或只要底层存储驱动程序允许）。

```php
$this->cache->set(
    key: 'key', 
    value: ['some' => 'data']
);
```

您还可以使用 `setMultiple` 方法并传入一个键/值对数组来一次存储多个项目。

```php
$this->cache->setMultiple(
    values: [
        'key' => ['some' => 'data'], 
        'other' => ['foo' => 'bar']
    ], 
    ttl: 3600
);
```

### 移除项目

如果您想从缓存中删除一个项目，您可以使用 `delete` 方法。

```php
$this->cache->delete('key');
```

您还可以使用 `deleteMultiple` 方法并传入一个键数组来一次删除多个项目。

```php
$this->cache->deleteMultiple(['key', 'other']);
```

### 清除缓存

要从缓存中删除所有项目，您可以使用 `clear` 方法。

```php
$this->cache->clear();
```

## 自定义存储

该组件允许通过配置文件集成 [自定义存储](https://packagist.org/providers/psr/simple-cache-implementation) 方法，使其具有通用性和易于使用。因此，您可以使用任何可用的 PSR-16 实现。

为了使用自定义存储，您需要在配置文件的 `storages` 部分中注册它，并使用 `type` 键指定存储类型，以及该存储的任何必要的配置选项。

您还需要在配置文件的 `typeAliases` 部分中注册该类，以便 Spiral 知道如何实例化您的自定义存储。

## 事件

| 事件                            | 描述                                                                 |
|----------------------------------|-----------------------------------------------------------------------------|
| `Spiral\Cache\Event\CacheHit`    | 当从缓存中成功检索到数据时将触发此事件。                                    |
| `Spiral\Cache\Event\CacheMissed` | 如果在缓存中未找到请求的数据，将触发此事件。                                   |
| `Spiral\Cache\Event\KeyDeleted`  | 在从缓存中删除数据 `后` 将触发此事件。                                      |
| `Spiral\Cache\Event\KeyWritten`  | 在将数据存储在缓存中 `后` 将触发此事件。                                     |

> **注意**
> 要了解有关分派事件的更多信息，请参阅我们文档中的 [事件](../advanced/events.md) 部分。
