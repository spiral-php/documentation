# HTTP — CSRF protection

Spiral provides built-in support for CSRF (Cross-Site Request Forgery) protection, making it easy for developers to 
implement this important security measure in their web applications and ensure that any actions taken on the website are 
intended by the user and not the result of a malicious attack.

Spiral uses cookies to store the CSRF token and does not rely on server-side sessions. This approach is considered to
be more efficient and simple, as it reduces the need for server-side storage and allows for faster performance.

## Vulnerability explanation

Let's imagine your application has a page to change the user's password and user can send a `POST` request to this page
with a new password. If application does not properly validate the authenticity of the user, an attacker may be able to
change the of user's password by tricking application into thinking the request is coming from the actual user. This can
be done via a malicious page.

**Here is an example of a malicious page:**

```html Malicious page
<form action="https://your-application.com/user/password" method="POST">
    <input name="password" type="password" value="secret">
</form>

<script>
    document.forms[0].submit();
</script>
```

Without CSRF protection, the user's password will be changed when a user visits a page with malicious code.

To prevent this vulnerability, it's important to implement proper CSRF protection in your application. One way to do
this is by inspecting every incoming `POST`, `PUT`, `PATCH`, or `DELETE` request for a CSRF token value that the
malicious website is unable to access. The token often referred to as a CSRF token, can be generated on the server and
included as a hidden field in forms.

```html
<form action="https://your-application.com/user/password" method="POST">
    <input type="hidden" name="csrf-token" value="{csrfToken}"/>
    <input name="password" type="password">
    // ...
    <button type="submit">Change password</button>
</form>
```

When the user submits a form, the server can then compare the value of the CSRF token included in the request to the
value stored in user cookie. If the values do not match, the server can reject the request and prevent the malicious
action from being performed.

## Configuration

The default `spiral/app` includes CSRF protection middleware.

To install it in alternative application:

Add `Spiral\Bootloader\Http\CsrfBootloader` to the list of bootloaders:

:::: tabs

::: tab Using method

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

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Http\CsrfBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

After the bootloader is added you need to enable `Spiral\Csrf\Middleware\CsrfMiddleware` middleware to issue a unique
token for every user.

To enable the middleware, add it to middleware group you want to protect:

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

> **See more**
> Read more about global middleware in the [HTTP — Middleware](middleware.md#global-middleware) section.

After the middleware is added, you may configure some options via `app/config/csrf.php`.

Here is the default configuration:

```php app/config/csrf.php
return [
    'cookie'   => 'csrf-token',
    'length'   => 16,
    'lifetime' => 86400,
    'secure'   => true,
    'sameSite' => null,
];
```

> **Warning**
> In you change `cookie` option, you must also add it to the whitelist cookie list.
> Read more how to do it in the [HTTP — Cookies](cookies.md#configuration) section.

## Enable Firewall

The component provides two middlewares which activate protection on your routes. 

To protect all the requests except for `GET`, `HEAD`, `OPTIONS `, use `Spiral\Csrf\Middleware\CsrfFirewall`:

```php app/src/Application/Bootloader/RoutesBootloader.php
use Spiral\Csrf\Middleware\CsrfFirewall;

'web' => [
    CookiesMiddleware::class,
    CsrfMiddleware::class,
    CsrfFirewall::class,
    // ...
],
```

> **Note**
> To protect against all the HTTP verbs, use `Spiral\Csrf\Middleware\StrictCsrfFirewall`.

## Usage

Once the protection firewall is activated, you must sign desired forms with the token available via PSR-7 
attribute `csrfToken`.

> **Note**
> `csrfToken` attribute is generated by `Spiral\Csrf\Middleware\CsrfMiddleware` middleware on every request.

To receive this token from the request in the controller or view use `getAttribute` method:

```php
public function index(ServerRequestInterface $request): void
{
    $csrfToken = $request->getAttribute('csrfToken');
}
``` 

Every `POST`/`PUT`/`DELETE` request from the user must include this token as POST parameter `csrf-token` or
header `X-CSRF-Token`. Users will receive `412 Bad CSRF Token` if a token is missing or not set.

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

You can also use view global variables to define `csrf-token` globally for all view templates.

### Registering a view global variable

Here is an example how to do it via middleware:

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

> **See more**
> Read more about global variables in the [Views — Basics](../views/basics.md#global-variables) section.

Don't forget to add the middleware to the middleware list.

```php app/src/Application/Bootloader/RoutesBootloader.php
'web' => [
    CookiesMiddleware::class,
    CsrfMiddleware::class,
    ViewCsrfTokenMiddleware::class,
    CsrfFirewall::class,
    // ...
],
```

After that, you can use the `csrfToken` variable in your views:

```html app/views/user/password.dark.php
<form action="https://your-application.com/user/password" method="POST">
    <input type="hidden" name="csrf-token" value="{csrfToken}"/>
    <input name="password" type="password">
    // ...
    <button type="submit">Change password</button>
</form>
```