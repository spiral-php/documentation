# 组件 — 事件 (Events)

Events 组件提供了一些工具，允许你的应用程序组件通过分发事件和监听事件来进行相互通信。该组件使得在你的应用程序中注册事件监听器变得非常容易。

它还提供了一种简便的方式来集成 `PSR-14` 兼容的 `EventDispatcher`。

## 安装

为了启用该组件，你需要将 `Spiral\Events\Bootloader\EventsBootloader` 添加到 bootloader 列表中，该列表位于你的应用程序的类中。

:::: tabs

::: tab 使用方法 (Using method)

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Events\Bootloader\EventsBootloader::class,
        // ...
    ];
}
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloader 的信息。
:::

::: tab 使用常量 (Using constant)

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Events\Bootloader\EventsBootloader::class,
    // ...
];
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloader 的信息。
:::

::::

这个 bootloader 在应用程序容器中注册了 `Spiral\Events\ListenerFactoryInterface` 和 `Spiral\Events\ListenerProcessorRegistry` 的实现。它还注册了应用程序中的事件监听器。我们将在下面描述如何添加它们。

之后，让我们安装一个与 `PSR-14` 兼容的 `EventDispatcher` 的实现。我们提供了一个 `spiral-packages/league-event` 包，该包将 PSR-14 兼容的 `league/event` 包集成到基于 Spiral 的应用程序中。

```terminal
composer require spiral-packages/league-event
```

安装完该包后，你需要注册来自该包的 bootloader。

:::: tabs

::: tab 使用方法 (Using method)

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Events\Bootloader\EventsBootloader::class,
        \Spiral\League\Event\Bootloader\EventBootloader::class,
        // ...
    ];
}
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloader 的信息。
:::

::: tab 使用常量 (Using constant)

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Events\Bootloader\EventsBootloader::class,
    \Spiral\League\Event\Bootloader\EventBootloader::class,
    // ...
];
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloader 的信息。
:::

::::

## 配置

Events 组件的配置文件位于 `app/config/events.php`。这个文件允许你指定事件 `listeners` 和 `processors`。

`listeners` 参数表示一个 `array`，键是 `事件类的完全限定名称`，而值必须是一个 `array`，其中包含 `事件监听器类的完全限定名称`，或者 `Spiral\Events\Config\EventListener` 实例，该实例允许你配置额外的监听器选项。

`processors` 参数表示一个 `array`，包含 `处理器类的全名`。

> **注意 (Note)**
> 关于 `processors` 的详细信息将在下面提供。本节仅描述配置。

例如，配置文件可能如下所示：

```php app/config/events.php
use App\Listener\RouteListener;
use Spiral\Events\Config\EventListener;
use Spiral\Events\Processor\AttributeProcessor;
use Spiral\Events\Processor\ConfigProcessor;
use Spiral\Router\Event\RouteMatched;

return [
    /**
     * -------------------------------------------------------------------------
     *  监听器 (Listeners)
     * -------------------------------------------------------------------------
     *
     * 在应用程序中注册的事件监听器列表。
     */
    'listeners' => [
        // 没有任何选项 (without any options)
        RouteMatched::class => [
            RouteListener::class,
        ],

        // 或者 (OR)

        // 使用附加选项 (with additional options)
        RouteMatched::class => [
            new EventListener(
                listener: RouteListener::class,
                method: 'onRouteMatched',
                priority: 1
            ),
        ],
    ],

    /**
     * -------------------------------------------------------------------------
     *  处理器 (Processors)
     * -------------------------------------------------------------------------
     *
     * 所有可用处理器的数组。
     */
    'processors' => [
        AttributeProcessor::class,
        ConfigProcessor::class,
    ],
];
```

## 使用 (Usage)

### 事件 (Event)

事件可以由一个简单的类来表示。

```php
namespace App\Event;

use App\Database\User;

final class UserWasCreated
{
    public function __construct(
        public readonly User $user
    ) {
    }
}
```

分发一个事件 (Dispatching an event)。

```php
namespace App\Service;

use Psr\EventDispatcher\EventDispatcherInterface;
use App\Event\UserWasCreated;

final class UserService
{
    public function __construct(
        private readonly EventDispatcherInterface $dispatcher
    ) {
    }

    public function create(string $username): User
    {
        $user = new User(username: $username);
        // ...
        $this->dispatcher->dispatch(new UserWasCreated($user));

        return $user;
    }
}
```

### 监听器 (Listener)

监听器可以由一个简单的类来表示，该类具有一个方法，该方法将被调用来处理事件。 方法名称可以在 Listener 属性参数或配置文件中配置（默认情况下为 `__invoke`）。

```php
namespace App\Listener;

use App\Event\UserWasCreated;
use Spiral\Events\Attribute\Listener;

#[Listener]
class UserWasCreatedListener
{
    public function __invoke(UserWasCreated $event): void
    {
        // ...
    }
}
```

使用该属性，你可以配置其他参数。

```php
namespace App\Listener;

use App\Event\UserWasCreated;
use Spiral\Events\Attribute\Listener;

#[Listener(event: UserWasCreated::class, method: 'onUserWasCreated', priority: 1)]
class UserWasCreatedListener
{
    public function onUserWasCreated(UserWasCreated $event): void
    {
        // ...
    }
}
```

该属性可以 `直接在方法上` 使用，然后可以省略方法名称。

```php
namespace App\Listener;

use App\Event\UserWasCreated;
use Spiral\Events\Attribute\Listener;

class UserWasCreatedListener
{
    #[Listener(event: UserWasCreated::class, priority: 1)]
    public function onUserWasCreated(UserWasCreated $event): void
    {
        // ...
    }
}
```

`Spiral\Events\Attribute\Listener` 属性用于自动在应用程序中注册事件监听器。
如果你更喜欢通过配置文件注册监听器，你可以删除此属性，以及配置文件中的 `Spiral\Events\Processor\AttributeProcessor`。

## 拦截器 (Interceptors)

Events 组件提供了拦截事件的能力。当你需要修改事件数据，或者例如通过 websockets 发送事件时，这很有用。为此，你需要创建一个拦截器类，该类实现 `Spiral\Core\CoreInterceptorInterface` 接口。

**例子 (Example)**

```php
namespace App\Broadcasting;

use Spiral\Broadcasting\BroadcastInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Queue\SerializerRegistryInterface;

final class BroadcastEventInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly BroadcastInterface $broadcast,
        private readonly SerializerRegistryInterface $registry
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        $event = $parameters['event'];

        // 首先分发事件 (Dispatch event first)
        $result = $core->callAction($controller, $action, $parameters);

        // 在分发后广播事件 (Broadcast event after dispatch)
        if ($event instanceof ShouldBroadcastInterface) {
            $this->broadcast->publish(
                $event->getBroadcastTopics(),
                $this->registry->getSerializer('json')->serialize(
                    ['event' => $event->getEventName(), 'data' => $event->getPayload()]
                )
            );
        }

        return $result;
    }
}
```

然后，你需要在 `events.php` 配置文件中注册它，该文件位于 `app/config` 目录中。

```php app/config/events.php
return [
    // ...
    'interceptors' => [
        \App\Broadcasting\BroadcastEventInterceptor::class
    ]
];
```

> **另请参阅 (See more)**
> 在 [框架 — 拦截器 (Framework — Interceptors)](../framework/interceptors.md) 部分阅读更多关于拦截器的信息。

## 创建一个事件分发器 (Creating an Event dispatcher)

作为 Event 分发器的实现，我们考虑了包
[The League Event bridge for Spiral](https://github.com/spiral-packages/league-event)。
该包提供了 Spiral 和 [The League Event](https://event.thephpleague.com/3.0/) 包之间的桥梁。

你可以使用 `PSR-14` 事件分发器创建你自己的实现。

### 监听器注册器 (ListenerRegistry)

让我们创建一个类，该类将实现 `Spiral\Events\ListenerRegistryInterface` 并提供注册事件监听器的能力。 该类中的 `addListener` 方法由 `processors` 调用，并将 `event`、`event listener` 和 `priority` 作为参数传递。

```php
namespace App\EventDispatcher;

use Spiral\Events\ListenerRegistryInterface;
use Psr\EventDispatcher\ListenerProviderInterface;

final class ListenerRegistry implements ListenerRegistryInterface, ListenerProviderInterface
{
    public function addListener(string $event, callable $listener, int $priority = 0): void
    {
        // ...
    }

    public function getListenersForEvent(object $event): iterable
    {
        // ...
    }
}
```

### 事件分发器 (Event dispatcher)

创建一个 `Psr\EventDispatcher\EventDispatcherInterface` 的实现。

```php
namespace App\EventDispatcher;

use Psr\EventDispatcher\EventDispatcherInterface;

final class EventDispatcher implements EventDispatcherInterface
{
    public function dispatch(object $event)
    {
        // ...
    }
}
```

### Bootloader

之后，我们需要创建一个 bootloader，它将在应用程序容器中注册接口的实现。

```php
namespace App\EventDispatcher\Bootloader;

use App\EventDispatcher\ListenerRegistry;
use App\EventDispatcher\EventDispatcher;

use Psr\EventDispatcher\EventDispatcherInterface;
use Psr\EventDispatcher\ListenerProviderInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Events\Bootloader\EventsBootloader;
use Spiral\Events\ListenerRegistryInterface;

final class EventBootloader extends Bootloader
{
    protected const DEPENDENCIES = [
        EventsBootloader::class
    ];

    protected const SINGLETONS = [
        ListenerRegistryInterface::class => ListenerRegistry::class,
        ListenerRegistry::class => ListenerRegistry::class,
        ListenerProviderInterface::class => ListenerRegistry::class,
        EventDispatcherInterface::class => EventDispatcher::class,
        EventDispatcher::class => EventDispatcherInterface::class
    ];
}
```

> **注意 (Note)**
> 不要忘记在应用程序中注册 bootloader。

## 处理器 (Processors)

事件处理器允许你从应用程序内的各种来源（例如配置文件或 PHP 属性）注册事件监听器。这对于组织和集中管理应用程序中的事件监听器非常有用。

默认情况下，Events 组件提供了两个处理器，你可以使用它们在应用程序中注册事件监听器。

**这些处理器是：**

- `Spiral\Events\Processor\ConfigProcessor`，它允许你从配置文件中注册事件监听器。
- `Spiral\Events\Processor\AttributeProcessor`，它允许你使用 PHP 属性 `Spiral\Events\Attribute\Listener` 注册事件监听器。

> **注意 (Note)**
> 你可以使用提供的处理器，只使用其中一个，或根据需要添加你自己的自定义处理器。只需更新配置文件 `app/config/events.php` 以指定你想要使用的处理器。

### 创建一个处理器 (Creating a Processor)

处理器是一个类，它必须实现 `Spiral\Events\Processor\ProcessorInterface` 并实现 `process` 方法。

```php
namespace App\Processor;

use Spiral\Events\ListenerFactoryInterface;
use Spiral\Events\ListenerRegistryInterface;
use Spiral\Events\Processor\AbstractProcessor;

final class MyCustomProcessor extends AbstractProcessor
{
    public function __construct(
        private readonly ListenerFactoryInterface $factory,
        private readonly ?ListenerRegistryInterface $registry = null,
    ) {
    }

    public function process(): void
    {
        // 如果 EventDispatcher 实现未在应用程序中注册，
        // 则不会注册此接口实现，并且处理器将停止工作。
        if ($this->registry === null) {
            return;
        }

        // 使用 ListenerRegistryInterface 实现，我们可以注册事件监听器。
        $this->registry->addListener(
            event: $event,
            listener: $this->factory->create($listener, $method),
            priority: $priority
        );
    }
}
```

`Spiral\Events\ListenerFactoryInterface` 的实现允许你从完全限定的类名和方法名创建一个监听器实例，并将所有必要的依赖项添加到构造函数中。

之后，我们需要在配置文件中注册处理器。

```php app/config/events.php
use App\Processor\MyCustomProcessor;

return [
    // ...
    'processors' => [
        MyCustomProcessor::class,
        // ...
    ],
    // ...
];
```
