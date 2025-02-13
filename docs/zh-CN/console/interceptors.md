# 控制台 - Interceptor（拦截器）

Interceptor（拦截器）允许您拦截控制台命令的执行，并在命令执行前后执行一些逻辑。

## 创建一个 Interceptor（拦截器）

要创建 interceptor（拦截器），您需要创建一个类并实现 `Spiral\Core\CoreInterceptorInterface` 接口。

```php
namespace App;

use Spiral\Console\Command;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

class CustomInterceptor implements CoreInterceptorInterface
{

    /**
     * @param array{
     *     input: InputInterface, 
     *     output: OutputInterface, 
     *     command: Command
     * }|array<empty, empty> $parameters
     */
    public function process(
        string $commandClass, 
        string $method, 
        array $parameters, CoreInterface $core
    ): int {
        // ...

        $result = $core->callAction($commandClass, $method, $parameters);

        // ...

        return $result;
    }
}
```

## 注册一个新的 Interceptor（拦截器）

Interceptor（拦截器）必须在应用程序中注册才能正常工作。 有几种方法可以添加一个新的拦截器。

### 通过配置

将其添加到配置文件 `app/config/console.php` 中。

```php
use App\CustomInterceptor;
use Spiral\Core\Container\Autowire;

return [    
    /**
     * -------------------------------------------------------------------------
     *  拦截器列表
     * -------------------------------------------------------------------------
     */
    'interceptors' => [
        // 通过完全限定的类名
        CustomInterceptor::class,
        
        // 通过 Autowire
        new Autowire(CustomInterceptor::class),
    ],
];
```

### 通过 ConsoleBootloader（Console 启动程序）

在 `Spiral\Console\Bootloader\ConsoleBootloader` 类中调用 `addInterceptor` 方法。

```php
namespace App\Application\Bootloader;

use App\CustomInterceptor;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class AppBootloader extends Bootloader
{
    public function boot(ConsoleBootloader $console): void
    {
        $console->addInterceptor(CustomInterceptor::class);
    }
}
```
