# HTTP — CSRF 保护

Spiral 提供了内置的 CSRF（跨站请求伪造）保护，使开发者能够在他们的 Web 应用程序中轻松实现这一重要的安全措施，并确保在网站上执行的任何操作都是用户意图的，而不是恶意攻击的结果。

Spiral 使用 Cookie 存储 CSRF 令牌，并且不依赖于服务器端的会话。 这种方法被认为是更高效和简单的，因为它减少了对服务器端存储的需求，并允许更快的性能。

## 漏洞解释

让我们设想你的应用程序有一个用于更改用户密码的页面，并且用户可以向此页面发送包含新密码的 `POST` 请求。 如果应用程序没有正确验证用户的真实性，攻击者可能通过欺骗应用程序认为该请求来自实际用户，从而更改用户的密码。 这可以通过恶意页面来完成。

**这是一个恶意页面的示例：**

```html 恶意页面
<form action="https://your-application.com/user/password" method="POST">
    <input name="password" type="password" value="secret">
</form>

<script>
    document.forms[0].submit();
</script>
```

没有 CSRF 保护，当用户访问包含恶意代码的页面时，用户的密码将被更改。

为了防止此漏洞，在你的应用程序中实现适当的 CSRF 保护非常重要。 一种方法是检查每个传入的 `POST`、`PUT`、`PATCH` 或 `DELETE` 请求，以查找恶意网站无法访问的 CSRF 令牌值。 该令牌通常被称为 CSRF 令牌，可以在服务器上生成并作为隐藏字段包含在表单中。

```html
<form action="https://your-application.com/user/password" method="POST">
    <input type="hidden" name="csrf-token" value="{csrfToken}"/>
    <input name="password" type="password">
    // ...
    <button type="submit">Change password</button>
</form>
```

当用户提交表单时，服务器可以将请求中包含的 CSRF 令牌的值与用户 Cookie 中存储的值进行比较。 如果这些值不匹配，服务器可以拒绝该请求，并阻止执行恶意操作。

## 配置

默认的 `spiral/app` 包含了 CSRF 保护中间件。

要在备选应用程序中安装它：

将 `Spiral\Bootloader\Http\CsrfBootloader` 添加到引导程序列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Http\CsrfBootloader::class,
        // ...
    ];
}
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的内容。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Http\CsrfBootloader::class,
    // ...
];
```

在 [框架 — 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的内容。
:::

::::

添加引导程序后，你需要启用 `Spiral\Csrf\Middleware\CsrfMiddleware` 中间件，以便为每个用户颁发唯一的令牌。

要启用中间件，请将其添加到你要保护的中间件组：

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Csrf\Middleware\CsrfMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                CookiesMiddleware::class,
                CsrfMiddleware::class,
                // ...
            ],
            // ...
        ];
    }

    // ...
}
```

> **另请参阅**
> 在 [HTTP — 中间件](middleware.md#global-middleware) 部分阅读更多关于全局中间件的内容。

添加中间件后，你可以通过 `app/config/csrf.php` 配置一些选项。

这是默认配置：

```php app/config/csrf.php
return [
    'cookie'   => 'csrf-token',
    'length'   => 16,
    'lifetime' => 86400,
    'secure'   => true,
    'sameSite' => null,
];
```

> **警告**
> 如果你更改了 `cookie` 选项，则还必须将其添加到白名单 Cookie 列表中。
> 在 [HTTP — Cookies](cookies.md#configuration) 部分阅读更多关于如何操作的信息。

## 启用防火墙

该组件提供了两个在你的路由上激活保护的中间件。

要保护所有请求，除了 `GET`、`HEAD`、`OPTIONS`，使用 `Spiral\Csrf\Middleware\CsrfFirewall`：

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Csrf\Middleware\CsrfFirewall;

'web' => [
    CookiesMiddleware::class,
    CsrfMiddleware::class,
    CsrfFirewall::class,
    // ...
],
```

> **注意**
> 要防止所有 HTTP 动词，请使用 `Spiral\Csrf\Middleware\StrictCsrfFirewall`。

## 使用

激活保护防火墙后，你必须使用通过 PSR-7 属性 `csrfToken` 可用的令牌来签署所需的表单。

> **注意**
> `csrfToken` 属性由 `Spiral\Csrf\Middleware\CsrfMiddleware` 中间件在每个请求上生成。

要在控制器或视图中从请求中接收此令牌，请使用 `getAttribute` 方法：

```php
public function index(ServerRequestInterface $request): void
{
    $csrfToken = $request->getAttribute('csrfToken');
}
```

来自用户的每个 `POST`/`PUT`/`DELETE` 请求都必须包含此令牌作为 POST 参数 `csrf-token` 或标头 `X-CSRF-Token`。 如果缺少令牌或未设置令牌，用户将收到 `412 Bad CSRF Token`。

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function changePasswordForm(ServerRequestInterface $request): string
{
    $form = <<<FORM
<form action="https://your-application.com/user/password" method="POST">
    <input type="hidden" name="csrf-token" value="{csrfToken}"/>
    <input name="password" type="password">
    // ...
    <button type="submit">Change password</button>
</form>
FORM;

    return \str_replace(
        '{csrfToken}',
        $request->getAttribute('csrfToken'),
        $form
    );
}
```

你还可以使用视图全局变量为所有视图模板全局定义 `csrf-token`。

### 注册视图全局变量

这是一个通过中间件执行此操作的示例：

```php
use Psr\Http\Server\MiddlewareInterface;
use Spiral\Views\GlobalVariablesInterface ;

class ViewCsrfTokenMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly GlobalVariablesInterface $globalVariables
    ) {}
    
    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        $this->globalVariables->set('csrfToken', $request->getAttribute('csrfToken'));
        
        return $handler->handle($request)->withAddedHeader('My-Header', 'my-value');
    }
}
```

> **另请参阅**
> 在 [视图 — 基础](../views/basics.md#global-variables) 部分阅读更多关于全局变量的内容。

不要忘记将中间件添加到中间件列表中。

```php app/src/Application/Bootloader/RoutesBootloader.php
'web' => [
    CookiesMiddleware::class,
    CsrfMiddleware::class,
    ViewCsrfTokenMiddleware::class,
    CsrfFirewall::class,
    // ...
],
```

之后，你可以在视图中使用 `csrfToken` 变量：

```html app/views/user/password.dark.php
<form action="https://your-application.com/user/password" method="POST">
    <input type="hidden" name="csrf-token" value="{csrfToken}"/>
    <input name="password" type="password">
    // ...
    <button type="submit">Change password</button>
</form>
```
