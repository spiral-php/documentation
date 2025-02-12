# HTTP — Cookies

默认的应用骨架程序默认启用 cookie 集成。

如果你需要在其他构建环境中启用 cookies，请引入 Composer 包 `spiral/cookies`，并将引导程序 `Spiral\Bootloader\Http\CookiesBootloader` 添加到你的应用中。

## Cookie 管理器

在 Spiral 中管理 cookies 最简单的方法是获取 `Spiral\Cookies\CookieManager` 的实例。这个实例可以存储在单例服务和控制器中，并提供对活动请求范围的访问。

```php
public function index(CookieManager $cookies): void
{
    dump($cookies->getAll());
    $cookies->set('name', 'value'); // 阅读以下关于更多选项的内容
}
```

如果你使用 `spiral/prototype` 扩展，你也可以使用 `cookies` 原型属性来访问 `CookieManager`：

```php
use PrototypeTrait;

public function index(): void
{
    dump($this->cookies->getAll());
    $this->cookies->set('name', 'value'); // 阅读以下关于更多选项的内容
}
```

阅读以下有关底层 cookie 管理的更多信息。

## 读取 Cookie

默认情况下，框架将使用 ENV 密钥 `ENCRYPTER_KEY` 加密和解密所有 cookie 值。更改此值将自动使所有为所有用户设置的 cookie 值失效。

> **注意**
> 你也可以出于性能原因禁用加密，或者使用替代的 HMAC 签名方法（见下文）。

cookie 组件将解密所有值并更新请求对象，因此你可以使用默认的 PSR-7 `ServerRequestInterface` 获取所有 cookie 值：

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function index(ServerRequestInterface $request): void
{
    dump($request->getCookieParams());
}
```

或者，你可以使用 `Spiral\Http\Request\InputManager` 读取 cookie 值，该管理器会自动解析请求范围，并可以作为单例引用：

```php
class HomeController
{
    private $input;

    public function __construct(InputManager $input)
    {
        $this->input = $input;
    }

    public function index(): void
    {
        dump($this->input->cookies->all());
    }
}
```

> **注意**
> 你也可以在请求过滤器中请求 cookie 值。

请注意，如果 cookie 值无效或无法解密，其值将设置为 NULL，并且应用程序将无法使用该值。

## 写入 Cookie

由于所有 cookie 值都必须被加密或签名，因此你必须使用正确的方式来写入它们。使用上下文特定的对象 `Spiral\Cookies\CookieQuery`。

```php
public function index(CookieQuery $cookies): void
{
    $cookies->set('name', 'value');
}
```

该方法按以下顺序接受以下参数：

| 参数      | 类型   | 描述                                                                                                                                                                                                                                                                                                                                                                                                   |
|-----------|--------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Name      | string | cookie 的名称。                                                                                                                                                                                                                                                                                                                                                                                        |
| Value     | string | cookie 的值。该值存储在客户端的计算机上；请勿存储敏感信息。                                                                                                                                                                                                                                                                                                                                                   |
| Lifetime  | int    | Cookie 的生命周期。此值以秒为单位指定，并声明 cookie 相对于当前时间的过期时间段。                                                                                                                                                                                                                                                                                                                                 |
| Path      | string | 服务器上 cookie 可用的路径。如果设置为 ‘/’，则 cookie 将在整个域中可用。如果设置为 ‘/foo/’，则 cookie 仅在域的 /foo/ 目录和所有子目录（例如 /foo/bar/）中可用。默认值是设置 cookie 的当前目录。                                                                                                                                                                                                                    |
| Domain    | string | cookie 可用的域。要使 cookie 在 example.com 的所有子域上可用，则应将其设置为 ‘.example.com’。 不需要 ‘.’，但它使 cookie 与更多浏览器兼容。将其设置为 www.example.com 将使 cookie 仅在 www 子域中可用。有关详细信息，请参阅规范中的尾部匹配。                                                                                                                                                                          |
| Secure    | bool   | 指示 cookie 应该仅通过客户端的安全 HTTPS 连接传输。设置为 true 时，仅当存在安全连接时才会设置 cookie。在服务器端，程序员的责任是仅在安全连接上发送此类 cookie（例如，针对 $_SERVER["HTTPS"]）。                                                                                                                                                                                                  |
| HttpOnly  | bool   | 当为 true 时，cookie 将只能通过 HTTP 协议访问。该标志意味着 cookie 无法通过脚本语言（例如 JavaScript）访问。此设置可以有效地帮助减少通过 XSS 攻击进行的身份盗窃（尽管并非所有浏览器都支持它）。                                                                                                                                                                                            |

> **注意**
>  `Spiral\Cookies\CookieManager::set()` 具有相同的参数。

## 与单例一起使用

你不能将 `CookieQuery` 用作 `__construct` 参数。 队列仅在 CookieMiddleware 的 IoC 上下文中可用，必须直接从容器中请求（使用上面显示的方法注入）：

```php
$container->get(CookieQuery::class)->set($name, $value);
```

> **注意**
> 使用 `CookieQuery` 的最佳位置是控制器方法。

如果你已经可以访问 `ServerRequestInterface`，你也可以使用属性 `cookieQueue`：

```php
use Psr\Http\Message\ServerRequestInterface;

// ...

public function index(ServerRequestInterface $request): void
{
    $request->getAttribute('cookieQueue')->set('name', 'value');
}
```

## 手动设置 Cookie

你可以通过调用 `Psr\Http\Message\ResponseInterface` 的 `withAddedHeader` 来手动写入 cookie 标头：

```php
return $response->withAddedHeader('Set-Cookie', 'name=value');
```

> 确保将 cookie 添加到白名单中，否则 CookieMiddleware 不会允许它通过。

## 配置

你可以使用 `Spiral\Bootloader\Http\CookiesBootloader` 配置 `CookieQueue` 的行为。

要在你的引导程序中将 cookie 列入白名单（禁用保护）：

```php
public function boot(CookiesBootloader $cookies): void
{
    $cookies->whitelistCookie('CustomCookie');
}
```

要在 cookie 组件上执行更深入的配置，请在 `app/config` 目录中创建配置文件 `cookies.php`：

```php app/config/cookies.php
use Spiral\Cookies\Config\CookiesConfig;

return [
    // 默认情况下，所有 cookies 将被设置为 .domain.com
    'domain'   => '.%s',

    // 保护方法
    'method'   => CookiesConfig::COOKIE_ENCRYPT,

    // 白名单 cookie (不加密/解密)
    'excluded' => ['PHPSESSID', 'csrf-token']
];
```
