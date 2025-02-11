# 组件 — 序列化

Spiral 帮助你管理和组织代码。其中一个方面是序列化数据的能力，这涉及到将数据转换为可以存储或传输的格式，然后稍后进行反序列化，或转换回其原始形式。这在各种情况下都很有用，例如通过网络传输数据或将其存储在数据库中。

Spiral 默认包含一些基本的序列化工具，但这些工具可能并不总是满足更复杂的使用场景。在这种情况下，框架使得集成额外的序列化工具或开发自定义的应用程序内数据序列化解决方案变得容易。这允许你为你的特定需求选择最佳方法。

该组件默认在 [应用程序包](https://github.com/spiral/app) 中可用。

## 安装

要在你的 Spiral 应用程序中启用序列化组件，你需要将 `Spiral\Serializer\Bootloader\SerializerBootloader` 添加到启动器列表中。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Serializer\Bootloader\SerializerBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读关于启动器的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Serializer\Bootloader\SerializerBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读关于启动器的更多信息。
:::

::::

## 配置

序列化组件的配置存储在 `app/config/serializer.php` 文件中。在这个文件中，你可以配置一个可用的序列化器数组，并指定默认的序列化器。

例如，配置文件可能看起来像这样：

```php app/config/serializer.php
use Spiral\Core\Container\Autowire;
use Spiral\Serializer\Serializer\JsonSerializer;
use Spiral\Serializer\Serializer\PhpSerializer; 
use Spiral\Serializer\Serializer\CallbackSerializer;

return [
    /**
     * -------------------------------------------------------------------------
     *  默认序列化器
     * -------------------------------------------------------------------------
     *  已注册序列化器之一的键。
     */
    'default' => 'json',
    
    /**
     * -------------------------------------------------------------------------
     *  可用序列化器
     * -------------------------------------------------------------------------
     *  可用序列化器的列表。
     */
    'serializers' => [
        // 通过完全限定的类名
        'json' => JsonSerializer::class,
        
        // 通过 Autowire
        'serializer' => new Autowire(PhpSerializer::class),
        
        // 通过带参数的 Autowire
        'encrypted_serializer' => new Autowire(EncryptedPhpSerializer::class, ['secret' => env('ENCRYPTION_KEY')]),
        
        // 或手动实例化对象
        'callback' => new CallbackSerializer(
            serializeCallback: fn(mixed $payload): string => \json_encode($payload),
            unserializeCallback: fn(string|\Stringable $payload, string|object|null $type = null) => \json_decode($payload, true)
        )
    ],
];
```

## 可用的序列化器

该组件开箱即用了一些基本的序列化器，但也很容易集成额外的序列化工具或开发自定义解决方案以满足你的特定需求。

- `Spiral\Serializer\Serializer\JsonSerializer` (`json`) - 使用 PHP 函数 `json_encode` 和 `json_decode`。它不支持将数据水合到对象。
- `Spiral\Serializer\Serializer\PhpSerializer` (`serializer`) - 使用 PHP 函数 `serialize` 和 `unserialize`。
- `Spiral\Serializer\Serializer\ProtoSerializer` (`proto`) - 使用 Google Protobuf 进行序列化和反序列化。
- `Spiral\Serializer\Serializer\CallbackSerializer` - 使用回调函数进行序列化和反序列化。

还有一些软件包提供了额外的序列化器：

- [Symfony serializer](https://github.com/spiral-packages/symfony-serializer) - 使用 Symfony 序列化器组件。
- [Laravel Serializable Closure](https://github.com/spiral-packages/serializable-closure) - 使用 Laravel 可序列化闭包包。

## 使用

### SerializerInterface

可以使用 `Spiral\Serializer\SerializerInterface` 从容器中注入序列化器。它将引用默认的序列化器。

```php
namespace App\Service;

use Spiral\Serializer\SerializerInterface;

class MyService
{
    public function __construct(
        private readonly SerializerInterface $serializer,
    ) {
    }

    public function someMethod(): void
    {
        // 序列化
        $serialized = $this->serializer->serialize(['some' => 'data']);
        
        // 反序列化
        $array = $this->serializer->unserialize($serialized);
    }
}
```

### SerializerManager

你可以使用 `Spiral\Serializer\SerializerManager` 通过其配置中的字符串键获取特定的序列化器。

```php
namespace App\Service;

use Spiral\Serializer\SerializerManager;

class MyService
{
    public function __construct(
        private readonly SerializerManager $serializer,
    ) {
    }

    public function someMethod(): void
    {
        $serialized = $this->serializer->getSerializer('json')->serialize(['some' => 'data']);
        $array = $this->serializer->getSerializer('json')->unserialize($serialized);

        $serialized = $this->serializer->getSerializer('serializer')->serialize(['some' => 'data']);
        $array = $this->serializer->getSerializer('serializer')->unserialize($serialized);
    }
}
```

## 创建一个序列化器

### 序列化器类

要在 Spiral 中创建一个自定义序列化器，你需要实现 `Spiral\Serializer\SerializerInterface`。此接口定义了你的序列化器类必须实现的两个方法：`serialize(...)` 和 `unserialize(...)`。

这是一个实现 `SerializerInterface` 的自定义序列化器类的示例：

```php
<?php

declare(strict_types=1);

namespace App\Application\Serializer;

use Google\Protobuf\Internal\Message;
use Spiral\Serializer\SerializerInterface;

final class ProtoSerializer implements SerializerInterface
{
    public function serialize(mixed $payload): string|\Stringable
    {
        \assert($payload instanceof Message);

        return $payload->serializeToString();
    }

    public function unserialize(\Stringable|string $payload, object|string|null $type = null): mixed
    {
        \assert(
            $type !== null
            && \class_exists($type)
            && \is_a($type, Message::class, true),
        );

        $object = new $type();
        $object->mergeFromString((string)$payload);

        return $object;
    }
}
```

你的自定义序列化器类的 `serialize` 方法应该接受一个参数 `$payload`，它表示要序列化的数据。该方法应该将序列化的数据作为字符串返回。

你的自定义序列化器类的 `unserialize` 方法应该接受两个参数：`$payload`，它表示序列化的数据作为字符串，以及 `$type`，这是一个可选参数，用于指定应将有效负载反序列化为的类或对象的名称。此方法应返回反序列化的数据。

### 注册一个新的序列化器

有两种方法可以注册一个自定义序列化器：

:::: tabs
::: tab 注册表
使用 `Spiral\Serializer\SerializerRegistryInterface`：

```php
namespace App\Application\Bootloader;

use App\Application\Serializer\ProtoSerializer;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Serializer\SerializerRegistryInterface;

class SerializerBootloader extends Bootloader
{
    public function boot(SerializerRegistryInterface $registry): void
    {
        $registry->register('proto', new ProtoSerializer());
    }
}
```
:::

::: tab 配置
使用配置文件：

```php app/config/serializer.php
use App\Application\Serializer\ProtoSerializer;

return [
    'serializers' => [
        'proto' => ProtoSerializer::class,
        // 其他序列化器
    ],
];
```
:::
::::
