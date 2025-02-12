# Getting started — First HTTP controller

如果你正在使用 Spiral 来开发 PHP 项目，并且准备开始设置你的第一个控制器，这里有几个需要遵循的基本步骤：

## 创建一个控制器

首先，你需要创建一个控制器。在 Spiral 中，控制器是一个类，它定义了应用程序针对一组特定路由的行为。它负责处理传入的 [PSR-7](https://www.php-fig.org/psr/psr-7/) 兼容的请求，处理数据，然后向客户端返回一个响应。

为了轻松地创建你的第一个控制器，使用脚手架命令：

```terminal
php app.php create:controller CurrentDate
```

> **注意**
> 在 [Basics — Scaffolding](../basics/scaffolding.md#http-controller) 部分阅读更多关于脚手架的信息。

执行此命令后，以下输出将确认创建成功：

```output
Declaration of ' [32mCurrentDateController [39m' has been successfully written into ' [33mapp/src/Endpoint/Web/CurrentDateController.php [39m'.
```

现在，让我们将一些逻辑注入到我们新创建的控制器中。

这是一个返回当前日期和时间的控制器示例：

```php app/src/Endpoint/Web/CurrentDateController.php
namespace App\Endpoint\Web;

final class CurrentDateController 
{
    public function show(): string
    {
        return \date('Y-m-d H:i:s');
    }
}
```

下一步是将一个路由与你的控制器关联起来。

## 创建一个路由

:::: tabs

::: tab Using Attributes

Spiral 通过使用 PHP 属性简化了路由定义。你只需要将 `#[Route]` 属性添加到控制器的某个方法上，如下所示：

```php app/src/Endpoint/Web/CurrentDateController.php
use Spiral\Router\Annotation\Route;

// ...

#[Route(route: '/date', name: 'current-date', methods: 'GET')]
public function show(): string
{
    return \date('Y-m-d H:i:s');
}
```

:::

::: tab Using RoutingConfigurator

对于希望以一种方便且有组织的方式定义应用程序路由的开发者，Spiral 提供了在 `App\Application\Bootloader\RoutesBootloader` 类中的 `defineRoutes` 方法。

这是一个如何定义将由我们的控制器处理的路由的示例：

```php app/src/Application/Bootloader/RoutesBootloader.php
final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add(name: 'current-date', pattern: '/date')
            ->action(controller: CurrentDateController::class, action: 'show');
    }
}
```

:::

::::

要查看路由列表，请使用以下命令：

```terminal
php app.php route:list
```

你应该在显示的列表中看到你的 `current-date` 路由：

```output
+--------------+--------+----------+------------------------------------------------+--------+
| [32m Name:         [39m| [32m Verbs:  [39m| [32m Pattern:  [39m| [32m Target:                                         [39m| [32m Group:  [39m|
+--------------+--------+----------+------------------------------------------------+--------+
| current-date |  [32mGET [39m    | /date    | App\Endpoint\Web\CurrentDateController->show | web    |
+--------------+--------+----------+------------------------------------------------+--------+
```

## 测试你的控制器

一旦你的控制器设置完毕，就可以通过使用以下命令运行 RoadRunner 服务器来测试它：

```terminal
./rr serve
```

现在，你可以在浏览器中访问该路由来测试你的控制器。只需在你的浏览器中打开以下 URL：http://127.0.0.1/date

<br><br>

**就是这样！你已经成功地在 Spiral 中设置了你的第一个控制器。**

<hr>

## 接下来呢？

现在，通过阅读一些文章来深入了解基础知识：

* [Routing](../http/routing.md)
* [Annotated Routing](../http/annotated-routes.md)
* [Middleware](../http/middleware.md)
* [Error Pages](../http/errors.md)
* [Custom HTTP handler](../cookbook/psr-15.md)
* [Scaffolding](../basics/scaffolding.md)
