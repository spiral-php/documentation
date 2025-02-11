# 框架 — 拦截器

Spiral 的关键特性之一是其对拦截器的支持，拦截器可用于为应用程序添加功能，而无需修改应用程序的核心代码。这有助于保持你的代码库更加模块化和可维护。

**使用拦截器的一些好处如下：**

- **关注点分离：** 使用拦截器可以让你将应用程序的不同部分分离并组织起来。例如，你可以使用拦截器来处理身份验证，而无需将该代码添加到需要身份验证的应用程序的每个单独部分。这使得理解和维护你的代码变得更加容易。
- **可重用性：** 通过拦截器，你可以编写一次代码并在应用程序的多个部分中使用它。这意味着你不必一遍又一遍地编写相同的代码，从而节省时间并降低出错的可能性。
- **模块化：** 在不影响应用程序其余部分的情况下添加、删除或替换拦截器的能力使其更灵活且易于更新。
- **性能：** 拦截器可用于通过缓存响应、减少数据库查询次数等方式优化应用程序的性能。这意味着你的应用程序会更快，并且在大量用户同时访问时不太可能变慢。
- **易用性：** 将拦截器添加到你的应用程序相对容易和直接，使其对所有技能水平的开发人员都可用。

你可以将拦截器与各种组件一起使用，例如：

- [HTTP](../http/interceptors.md)
- [事件](../advanced/events.md#interceptors)
- [gRPC](../grpc/interceptors.md)
- [Websocket](../websockets/interceptors.md)
- [队列](../queue/interceptors.md)

> **注意**
> 域核心需要 `spiral/hmvc` 组件。Web 包默认包含此包。

## 域核心

为了使用拦截器，你需要有一个核心类，该类负责运行可以在调用之前或之后被拦截的特定逻辑。

例如，假设你有一个数据库层，你需要记录数据库查询、测量查询时间以及记录慢查询。

为了实现这一点，你可以创建一个 `DatabaseQueryCore` 类，该类实现 `Spiral\Core\CoreInterface`。这个类将负责运行数据库查询并返回查询结果。

```php app/src/Integration/Database/DatabaseQueryCore.php
namespace App\Integration\Database;

use Spiral\Core\CoreInterface;
use Cycle\Database\DatabaseManager;
use Cycle\Database\StatementInterface;

final class DatabaseQueryCore implements CoreInterface
{
    public function __construct(
        private readonly DatabaseManager $manager
    ) {
    }

    public function callAction(string $database, string $sql, array $parameters = []): StatementInterface
    {
        $sqlParameters = $parameters['sql_parameters'] ?? [];
        \assert(\is_array($sqlParameters));

        $database = $this->manager->database($database);
        return $database->query(
            $sql,
            $sqlParameters
        );
    }
}
```

然后你可以使用 `Spiral\Core\InterceptableCore` 类，该类允许你注册拦截器，并在你处理核心类时调用它们。

```php
use Spiral\Core\InterceptableCore;
use App\Application\Database\DatabaseQueryCore;
use Cycle\Database\DatabaseManager;

$core = new InterceptableCore(
  new DatabaseQueryCore(new DatabaseManager(...))
);
```

下面是一个如何调用 `DatabaseQueryCore` 类来处理数据库查询的例子：

```php
// Execute a SELECT statement on the 'default' database
$result = $core->callAction(
  'default', 
  'SELECT * FROM users WHERE id = ?', 
  ['sql_parameters' => [1]]
);
```

`callAction` 方法将返回一个 `Cycle\Database\StatementInterface` 对象，该对象表示查询的结果。

现在我们来讨论一下拦截器。

## 拦截器

拦截器的工作方式与 HTTP 请求中的中间件类似，它们允许开发人员在处理流程的各个点为应用程序添加功能。但是，与通常特定于 HTTP 请求的中间件不同，拦截器可用于为各种组件添加功能。

拦截器应该实现 `Spiral\Core\CoreInterceptorInterface` 接口，该接口要求它们定义一个 `process` 方法。框架会在应用程序执行的特定点调用此方法，例如在调用控制器操作之前或之后。

例如，我们可以创建一个 `SlowQueryDetectorInterceptor` 类，该类实现了 `CoreInterceptorInterface` 并记录慢查询。

```php app/src/Integration/Database/Interceptor/SlowQueryDetectorInterceptor.php
namespace App\Integration\Database\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;

class SlowQueryDetectorInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    public function process(string $database, string $sql, array $parameters, CoreInterface $core): mixed
    {
        $startTime = \microtime(true);

        $result = $core->callAction($database, $sql, $parameters);
        
        $elapsed = \microtime(true) - $startTime;

        if ($elapsed > 0.1) {
            $this->logger->warning(
                'Slow query detected',
                [
                    'database' => $database,
                    'sql' => $sql,
                    'parameters' => $parameters,
                    'elapsed' => $elapsed
                ]
            );
        }

        return $result;
    }
}
```

现在我们可以使用 `InterceptableCore` 类注册 `SlowQueryDetectorInterceptor` 类，并调用 `callAction` 方法来处理数据库查询。

```php
use Spiral\Core\InterceptableCore;
use App\Application\Database\DatabaseQueryCore;
use Cycle\Database\DatabaseManager;

$core = new InterceptableCore(
  new DatabaseQueryCore(new DatabaseManager(...))
);

$core->addInterceptor(new SlowQueryDetectorInterceptor(new Logger(...)));

// Execute a SELECT statement on the 'default' database
$result = $core->callAction(
  'default', 
  'SELECT * FROM users WHERE id = ?', 
  ['sql_parameters' => [1]]
);
```

现在，每次调用 `callAction` 方法时，都会调用 `SlowQueryDetectorInterceptor` 类并记录慢查询。

我们还可以向 `InterceptableCore` 类添加多个拦截器。例如，我们可以创建另一个拦截器来处理数据库异常，并在某些情况下尝试重新连接到数据库。

```php app/src/Integration/Database/Interceptor/DatabaseConnectionInterceptor.php
namespace App\Integration\Database\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Cycle\Database\Exception\StatementException\ConnectionException;

final class DatabaseConnectionInterceptor implements CoreInterceptorInterface
{
    // ...

    public function process(string $database, string $sql, array $parameters, CoreInterface $core): mixed
    {
        try {
            return $core->callAction($database, $sql, $parameters);
        } catch (ConnectionException $e) {
            // Try to reconnect...

            // For example, switch to another database...
            return $this->process('slave', $sql, $parameters, $core);
        }
    }
}
```

不要忘记使用 `InterceptableCore` 类注册 `DatabaseConnectionInterceptor` 类。

```php
$core->addInterceptor(new DatabaseConnectionInterceptor(...));
$core->addInterceptor(new SlowQueryDetectorInterceptor(new Logger(...)));
```

你可以将任何参数传递给 `callAction` 方法，这些参数可以被拦截器使用。例如：

```php
public function process(string $database, string $sql, array $parameters, CoreInterface $core): mixed
{
    // $sql = SELECT * FROM users WHERE id = ?
    $sql .= ' LIMIT ?';
    
    $parameters['sql_parameters'][] = 10;
    
    return $core->callAction($database, $sql, $parameters);
}
```

正如你所看到的，拦截器是一种方便的方式来拦截和修改应用程序某些部分的行为，使其更具功能性和效率，同时保持特定于域的逻辑清晰和可维护。

## 事件

| 事件                                | 描述                                                       |
|--------------------------------------|-----------------------------------------------------------|
| Spiral\Core\Event\InterceptorCalling | 在调用拦截器`之前`将触发该事件。 |

> **注意**
> 要了解有关分发事件的更多信息，请参阅我们文档中的[事件](../advanced/events.md)部分。
