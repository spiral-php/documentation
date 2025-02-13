# 基础知识 — 会话 (Session)

如果你需要在其他的包中启用会话，请引入 Composer 包 `spiral/session`，并在你的应用中添加引导程序 `Spiral\Bootloader\Http\SessionBootloader`。

## SessionInterface (会话接口)

用户会话可以使用上下文相关的对象 `Spiral\Session\SessionInterface` 来访问：

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Session\SessionInterface;

// ...

public function index(SessionInterface $session): void
{
    $session->resume();
    dump($session->getID());
}
```

> **注意**
> 你不允许在单例对象中存储会话引用。请参考下面的解决方法。

## Session Section (会话分区)

默认情况下，你不能直接操作会话，而是分配一个隔离的、命名的分区，它提供了类似 `set`, `get`, `delete` 或任何其他功能。使用会话对象的 `getSection` 方法来实现这些目的：

```php app/src/Endpoint/Web/HomeController.php
public function index(SessionInterface $session): void
{
    $cart = $session->getSection('cart');

    $cart->set('items', ['my-items']);

    dump($cart->getAll());
}
```

## Session Scope (会话作用域)

为了简化在单例服务和控制器中使用会话，请使用 `Spiral\Session\SessionScope`。这个组件也可以通过原型属性 `session` 来访问。该组件可以在单例服务中使用，并且始终指向一个活动会话上下文：

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): void
    {
        dump($this->session->getSection('cart')->getAll());
    }
}
```

## Session Lifecycle (会话生命周期)

会话将在首次访问数据时自动启动，并在请求离开 `SessionMiddleware` 时提交。要手动控制会话，请使用 `Spiral\Session\SessionInterface` 对象的方法。

> **注意**
> SessionScope 完全实现了 SessionInterface。

### Resume session (恢复会话)

手动恢复/创建会话：

```php
$this->session->resume();
```

### Commit (提交)

手动提交并关闭会话：

```php
$this->session->commit();
```

### Abort (中止)

放弃所有更改并关闭会话：

```php
$this->session->abort();
```

### Get Session ID (获取会话 ID)

获取会话 ID（仅当会话已恢复时）：

```php
dump($this->session->getID());
```

检查会话是否已启动：

```php
dump($this->session->isStarted());
```

### Destroy (销毁)

销毁会话及其所有内容：

```php
$this->session->destroy();
```

### Regenerate ID (重新生成 ID)

颁发一个新的会话 ID，而不会影响会话内容：

```php
$this->session->regenerateID();
```

## Custom Configuration (自定义配置)

要修改会话配置，请创建文件 `app/config/session.php` 来更改所需的值。

会话组件基于原生的 PHP 会话实现。默认情况下，会话内容存储在 `runtime/session` 目录中的文件系统中。如果你的应用程序将在多个 Web 服务器之间进行负载均衡，则应该选择一个所有服务器都可以访问的集中式存储，例如 Redis。

会话 `handler` 配置选项定义了每次请求会话数据将存储在哪里。Spiral 提供了开箱即用的几个驱动程序：

### **FileHandler (文件处理器)** 配置

会话存储在 `runtime/session` 文件夹中。

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\FileHandler;

return [
    'lifetime' => 86400,
    'cookie' => 'sid',
    'secure' => false,
    'handler' => new Autowire(
        FileHandler::class,
        [
            'directory' => directory('runtime') . 'session',
            'lifetime'  => 86400
        ]
    )
];
```

### **CacheHandler (缓存处理器)** 配置

会话存储在 Cache 组件中配置的其中一个基于缓存的存储中。

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\CacheHandler;

$ttl = 86400;

return [
    'lifetime' => $ttl,
    'cookie' => 'sid',
    'secure' => false,
    'handler' => new Autowire(
        CacheHandler::class,
        [
            'storage' => 'my-storage', // (Optional)  Cache storage name. Default - current cache storage
            'ttl' => $ttl,
            'prefix' => 'foo:' // (Optional) By default, session:
        ]
    )
];
```

### Custom Session Handler (自定义会话处理器)

如果现有的任何会话驱动程序都不适合你的应用程序需求，Spiral 允许你编写自己的会话处理器。你的自定义会话驱动程序应实现 PHP 内置的 [`SessionHandlerInterface`](https://www.php.net/manual/en/class.sessionhandlerinterface.php)。

```php app/config/session.php
return [
    'handler' => new Autowire(
        MemoryHandler::class,
        [
            'driver' => 'redis',
            'database' => 1,
            'lifetime' => 86400
        ]
    )
];
```

> **注意**
> 你可以使用 Autowire 而不是类名来配置其他参数。

### Customize Session initialization (自定义会话初始化)

会话使用一个特殊的工厂 `Spiral\Session\SessionFactoryInterface` 进行初始化。

```php
namespace Spiral\Session;

interface SessionFactoryInterface
{
    /**
     * @param string $clientSignature User specific token, does not provide full security but
     *                                     hardens session transfer.
     * @param string|null $id When null - expect php to create session automatically.
     */
    public function initSession(string $clientSignature, string $id = null): SessionInterface;
}
```

你可以在容器中用你自己的实现替换 `Spiral\Session\SessionFactoryInterface` 的默认实现。

```php
$container->bindSingleton(\Spiral\Session\SessionFactoryInterface::class, CustomSessionFactory::class);
```
