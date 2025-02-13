# Framework — Finalizers

大多数框架组件不需要在请求完成之后进行任何重置。但是，当您希望在用户请求
完成后重置您的库时，存在多种用例。

> **注意**
> 优先使用 IoC 作用域而不是终结器。

## FinalizerInterface

使用 `Spiral\Boot\FinalizerInterface`

```php
/**
 * Finalizers 用于关闭长时间运行的进程的资源和连接。
 */
interface FinalizerInterface
{
    /**
     * 终结器会在每次请求之后执行，并用于垃圾回收
     * 或者关闭打开的连接。
     *
     * @param callable $finalizer
     */
    public function addFinalizer(callable $finalizer);
    
    /**
     * 终结执行。
     *
     * @param bool $terminate 如果在应用程序终止时触发，则设置为 true。
     */
    public function finalize(bool $terminate = false);
}
```

所有应用程序调度器都会调用终结器。 在以下情况下：

* HTTP 请求完成
* HTTP 请求因错误而失败
* job 已完成
* job 因错误而失败
* GRPC 调用完成
* GRPC 调用因错误而失败
* 控制台命令已完成

> **警告**
> 只有在启动特定的调度器后，才会调用终结器。您可以自由地调用应用程序
> 命令和 HTTP 方法，而无需直接使用调度器，并且无需在每次请求后重置您的服务。

您的处理程序将接收到布尔值参数，它指示应用程序是否要在请求后终止。

> **注意**
> 避免在终结器中重置 IoC，因为它可能会导致一些单例服务缓存之前的服务
> 版本。

## 示例 Finalizer

我们可以使用终结器来演示如何在每次请求后关闭数据库连接。
如果您运行大量工作程序（或 lambda 函数），并且不希望使用所有数据库套接字，那么它会很有用。

```php
// 在 bootloader 中
use Spiral\Boot\FinalizerInterface;
use Psr\Container\ContainerInterface;
use Cycle\Database\DatabaseManager;

public function boot(FinalizerInterface $finalizer, ContainerInterface $container): void
{
    $finalizer->addFinalizer(function () use ($container) {
        /** @var DatabaseManager $dbal */
        $dbal = $container->get(DatabaseManager::class);
 
        foreach ($dbal->getDrivers() as $driver) {
            $driver->disconnect();
        }
    });
}
```

> **注意**
> 您可以在 [`spiral\cycle-bridge` 包](https://github.com/spiral/cycle-bridge/blob/master/src/Bootloader/DisconnectsBootloader.php) 中找到这样一个启动加载器，它
> 可用作 `Spiral\Cycle\Bootloader\DisconnectsBootloader`。
