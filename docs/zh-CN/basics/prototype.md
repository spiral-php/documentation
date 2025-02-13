# 基础 — 原型开发

原型开发是应用开发的一个阶段，在这个阶段，你勾勒出你的应用程序结构，并快速迭代你的类及其交互的基本设计。Spiral 提供了一个强大的扩展，它增强了这个原型开发过程，通过 AST 修改（也称为它为你编写代码）来加速应用程序服务、控制器、中间件和其他类的开发。该扩展包括针对最常见的框架组件和 Cycle 仓库的 IDE 友好型工具提示。

## 安装

确保将 `Spiral\Prototype\Bootloader\PrototypeBootloader` 添加到你的 App 类中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Prototype\Bootloader\PrototypeBootloader::class,
        // ...
    ];
}
```

在 [框架 — 引导程序](../framework/bootloaders.md) 章节阅读更多关于引导程序的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Prototype\Bootloader\PrototypeBootloader::class,
    // ...
];
```

在 [框架 — 引导程序](../framework/bootloaders.md) 章节阅读更多关于引导程序的信息。
:::

::::

现在你可以运行 `php app.php configure` 来为你的应用程序生成 IDE 自动补全助手。

## 原型属性的使用

### 原型开发

为了开始使用原型开发功能，你需要将 `Spiral\Prototype\Traits\PrototypeTrait` trait 添加到任何你想要进行原型开发的类中。这个 trait 就像一个神奇的助手，它帮助你的 IDE 找到并使用应用程序的其他部分，而无需设置很多细节。

![IDE 工具提示](https://user-images.githubusercontent.com/796136/67619538-8f0c8c80-f805-11e9-9cd8-0597133bf33a.gif)

**以下是更详细的关于如何将它添加到控制器类中的方法：**

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

当你将这个 trait 包含在你的类中时，你的 IDE 将开始建议你可以使用哪些服务或组件。例如，如果你的应用程序有一个用户管理系统，键入 `$this->users` 可能会提示你的 IDE 建议来自你的用户服务或仓库的方法。

这对于快速测试想法非常棒，因为你不需要设置应用程序各部分之间的所有正式连接（比如依赖注入）。

### 将原型开发转化为真正的代码

神奇属性很方便，但从长远来看，它们对于性能和清晰度来说并不是最好的。因此，一旦你对事情的运作方式感到满意，你就可以告诉 Spiral 用实际的代码替换这些魔力。

只需运行以下命令：

```terminal
php app.php prototype:inject -r
```

这告诉 Spiral: *"遍历我的类，查找这些神奇的属性，并将它们变成真实的、坚实的代码。"* 它将自动添加必要的依赖注入代码，让你的应用程序准备好用于实际使用。

> **注意**
> 使用 `-r` 标志从类中移除 `PrototypeTrait`。

**以下是运行命令后代码的样子：**

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use App\Database\Repository\UserRepository;
use Spiral\Views\ViewsInterface;

class HomeController
{
    public function __construct(
        private readonly ViewsInterface $views, 
        private readonly UserRepository $users
    ) {
    }

    public function index(): string
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

### 使用原型开发发现类

在某些情况下，你可能想找到所有使用原型属性的类。要查看所有类，只需运行以下命令：

```terminal
php app.php prototype:usage
```

它将输出一个类列表，如下例所示：

```terminal
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
| Class:                                                             | Property:            | Target:                                                      |
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
| App\Endpoint\Web\Controller\User\SetupPasswordAction               | userService          | App\Service\UserServiceInterface                             |
|                                                                    | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
| App\Endpoint\Web\Controller\User\SetupPasswordFormAction           | users                | App\Repository\UserRepositoryInterface                       |
|                                                                    | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
|                                                                    | request              | Spiral\Http\Request\InputManager                             |
|                                                                    | sessionErrors        | App\Application\HTTP\SessionErrorsInterface                  |
| App\Endpoint\Web\Controller\Auth\LoginFormAction                   | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
|                                                                    | request              | Spiral\Http\Request\InputManager                             |
|                                                                    | sessionErrors        | App\Application\HTTP\SessionErrorsInterface                  |
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
```

> **注意**
> 在所有 inject 完成后，你可以移除 `spiral/prototype` 扩展。

### 审查变更

在上一步之后，审查变更是一个好主意，以确保所有内容都按照你的预期进行了设置。原型开发工具很智能，但进行复查总是有益的。你可能需要对代码进行调整或优化。

一旦你对设置感到满意，你就可以使用原型开发工具帮助你创建的坚实基础来继续构建你的应用程序。

## 自定义属性

你可以使用 `Spiral\Prototype\Bootloader\PrototypeBootloader` 在你的引导程序中注册任意数量的原型属性：

```php
use Spiral\Prototype\Bootloader\PrototypeBootloader;

public function boot(PrototypeBootloader $prototype): void
{
    $prototype->bindProperty('myService', MyService::class);
}
```

> **注意**
> 你可以将这种方法与自动类发现结合起来，以更好地将领域层架构集成到你的开发过程中。

## 基于属性

或者，你可以使用属性来注册原型类和服务。在你要注入的类中使用属性 `Spiral\Prototype\Annotation\Prototyped`：

```php app/src/Domain/User/Service/UserService.php
namespace App\Domain\User\Service;

use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'userService')]
final class UserService
{
    // ...
}
```

确保运行 `php app.php update` 或 `php app.php prototype:dump` 来自动定位你的服务。

> **警告**
> 要将属性与接口一起使用，你需要启用对接口的搜索。要这样做，请阅读
> [配置监听器](../advanced/tokenizer.md#configuring-listeners) 部分。

## 可用快捷方式

要查看应用程序中所有已注册的快捷方式，请运行：

```terminal
php app.php prototype:list
```

### 可用快捷方式

有许多组件快捷方式可供你使用：

| 属性        | 组件                                                                                     |
|-------------|-----------------------------------------------------------------------------------------------|
| app         | App\App (或实现了 `Spiral\Boot\Kernel` 的类)                                                |
| classLocator| Spiral\Tokenizer\ClassesInterface                                                             |
| console     | Spiral\Console\Console                                                                        |
| container   | Psr\Container\ContainerInterface                                                              |
| db          | Cycle\Database\DatabaseInterface (`spiral/cycle-bridge` 包应该被安装)                          |
| dbal        | Cycle\Database\DatabaseProviderInterface (`spiral/cycle-bridge` 包应该被安装)                  |
| encrypter   | Spiral\Encrypter\EncrypterInterface                                                           |
| env         | Spiral\Boot\EnvironmentInterface                                                              |
| files       | Spiral\Files\FilesInterface                                                                   |
| guard       | Spiral\Security\GuardInterface                                                                |
| http        | Spiral\Http\Http                                                                              |
| i18n        | Spiral\Translator\TranslatorInterface                                                         |
| input       | Spiral\Http\Request\InputManager                                                              |
| session     | Spiral\Session\SessionScope                                                                   |
| cookies     | Spiral\Cookies\CookieManager                                                                  |
| logger      | Psr\Log\LoggerInterface                                                                       |
| logs        | Spiral\Logger\LogsInterface                                                                   |
| memory      | Spiral\Boot\MemoryInterface                                                                   |
| orm         | Cycle\ORM\ORMInterface (`spiral/cycle-bridge` 包应该被安装)                                  |
| paginators  | Spiral\Pagination\PaginationProviderInterface                                                 |
| queue       | Spiral\Queue\QueueInterface                                                                   |
| queueManager| Spiral\Queue\QueueConnectionProviderInterface                                                 |
| request     | Spiral\Http\Request\InputManager                                                              |
| response    | Spiral\Http\ResponseWrapper                                                                   |
| router      | Spiral\Router\RouterInterface                                                                 |
| server      | Spiral\Goridge\RPC (`spiral/roadrunner-bridge` 包应该被安装)                                 |
| snapshots   | Spiral\Snapshots\SnapshotterInterface                                                         |
| storage     | Spiral\Storage\StorageInterface                                                               |
| validator   | Spiral\Validation\ValidationInterface                                                         |
| views       | Spiral\Views\ViewsInterface                                                                   |
| auth        | Spiral\Auth\AuthScope                                                                         |
| authTokens  | Spiral\Auth\TokenStorageInterface                                                             |
| cache       | Psr\SimpleCache\CacheInterface                                                                |
| cacheManager| Spiral\Cache\CacheStorageProviderInterface                                                    |
