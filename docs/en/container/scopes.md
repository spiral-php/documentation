# Container — IoC Scopes

Building long-lasting applications requires proper management of context. In many systems, you can no longer treat user requests as global singletons stored across services.

This means you need to explicitly request context when processing user input.
Spiral provides an easy way to manage this using an IoC (Inversion of Control) container scopes.

Scopes allow you to create isolated contexts where you can redefine services and manage their lifecycle.

## How to Create an Isolated Scope

To create an isolated context, use the `Container::runScope()` method. The first argument is a `Scope` object that contains all the scope options, and the second is a function that will run inside this scope.
The result of this function is returned by `runScope()`.

For example, you can redefine custom bindings to the scope by passing them in the `Scope` object:

```php
$result = $container->runScope(
    new Scope(bindings: [
        LoggerInterface::class => FileLogger::class,
    ]),
    function () {
        // Your code here
    },
);
```

In this example, the `LoggerInterface` will be resolved inside the scope as `FileLogger`.


## How It Works

When you call `$container->runScope(new Scope(...), fn() => ...)`, a new container is created with its own bindings. The existing container becomes the parent of this new container.

The new container will be used inside the provided function and will be destroyed after the function completes.

Important points:
- Visibility: parent containers don’t know about their child containers. However, services in the parent container are accessible within children.
- Scopes can be named or anonymous.
    - The main global scope is always `root`.
    - Named scopes must have unique names in a hierarchy to avoid conflicts.  
      ![scopes-conflict](https://gist.github.com/user-attachments/assets/32f1ae89-9e35-4e7a-9e53-b3db15fee0ea)
    - Parallel scopes with the same name (like in coroutines) can exist and will have their own hierarchies.
- When exiting a scope, the associated container is destroyed.

### Dependency Resolution Order

When resolving dependencies inside an isolated scope:
1. The container first tries to find a binding in the current scope.
2. If the binding is not found, the container tries to find it in the parent scope, and so on up to the root container.
3. The instance is created in the scope where the binding was found. This means that dependencies for this instance are resolved within that same scope.


## Predefined Scopes

There are several predefined scopes in Spiral:

![spiral-scopes](https://gist.github.com/user-attachments/assets/aa12be0a-bea1-439c-a676-ef8d6158bda9)

1. `root` — the main global scope. All other scopes are its children.
2. Dispatcher Scope — a scope that opens when the corresponding [Dispatcher](../framework/dispatcher.md) is started:
   `http`, `console`, `grpc`, `centrifugo`, `tcp`, `queue`, or `temporal`.
3. Request Scope — a scope that opens before the controller is executed, when the request object is fully formed and ready for processing.
   In the case of the HTTP dispatcher, middleware will be executed in the `http` scope, and interceptors in `http-request`.

If you are sure that a service will only work within a specific dispatcher, it makes sense to use the corresponding scope.
For example, HTTP middleware should be bound at the `http` scope level.

You can create your own scopes to isolate context and make only specific services available.

![http-scopes](https://gist.github.com/user-attachments/assets/a402f166-4396-40ec-a376-d2136fb25824)

## Configuring Bindings for Named Scopes

You can preconfigure bindings specific to named scopes using the `BinderInterface::getBinder()` method.
This allows you to set default bindings for a scope.

Example:

```php
$container->bindSingleton(Interface::class, Implementation::class);

// Configure default bindings for 'request' scope
$binder = $container->getBinder('request');
$binder->bindSingleton(Interface::class, Implementation::class);
$binder->bind(Interface::class, factory(...));
```

> Note:
> Bindings in a scope do not affect existing containers of that scope (except for `root`).


## Overriding Default Bindings

When using `Container::runScope()`, you can pass bindings to override defaults for a specific scope.

Example:

```php
$container->bindSingleton(SomeInterface::class, SomeImplementation::class);

$container->runScope(
    new Scope(
        name: 'request',
        bindings: [SomeInterface::class => AnotherImplementation::class],
    ),
    function () {
        // Your code here
    }
);
```

Here, even if the `request` scope has a default binding for `SomeInterface`, this specific run uses `AnotherImplementation`.


## Scope Restrictions

You can restrict where a dependency can be resolved using the `#[Scope('name')]` attribute.

Example:

```php
use Spiral\Boot\Environment\DebugMode;
use Spiral\Core\Attribute\Scope;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
#[Scope('http')]
final readonly class DebugMiddleware implements \Psr\Http\Server\MiddlewareInterface
{
    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // ...
    }
}
```

Here, the `DebugMiddleware` can be instantiated only if the `http` scope exists in the scope hierarchy.
Otherwise, an exception is thrown.


## Destroying Scopes and Finalization

When exiting a scope, the associated container is destroyed.
This means singletons created within the scope should be garbage collected, so avoid circular references.

If you need to perform cleanup actions when dependencies are resolved in a scope, use the `#[Finalize('methodName')]` attribute to specify a method that will be called when the scope is destroyed.

```php
#[Finalize('destroy')]
class MyService
{
    /**
     * This method will be called before the scope is destroyed in case the service was resolved in this scope.
     * Arguments will be resolved using the container.
     */
    public function destroy(LoggerInterface $logger): void
    {
        // Clean up...
    }
}
```


## Proxy Objects

Scopes are like nested containers, but there's more to them than simple delegation.

What if you want to create a stateless service in the parent scope (`root` or `http`)
that will handle `ServerRequestInterface` objects in the `http-request` scope?
With nested containers, this is impossible because `ServerRequestInterface` is only available inside the `http-request` scope.
Moreover, `ServerRequestInterface` will be different for each request.

Spiral provides proxy objects that defer dependency resolution until it’s actually needed.

Use the `#[Proxy]` attribute to create proxies for interfaces:

```php
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Core\Attribute\Proxy;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final readonly class DebugService
{
    public function __construct(
        #[Proxy] private ServerRequestInterface $request,
    ) {}

    public function hasDebugInfo(): bool
    {
        return $this->request->hasHeader('X-Debug');
    }
}
```

Important points:
- Proxies are configured only for interfaces.
- Each method call on a proxy will resolve the real object from the container.
- Calling methods not defined in the interface is disallowed.

You can configure proxies for services that must only be available in specific scopes using the `Binder` class.
For example, if the `AuthInterface` service must only be available in the `http` scope, you can use a proxy object for the `root` scope:

```php
// Configure a proxy for `AuthInterface` in the `root` scope
$rootBinder = $container->getBinder('root');
$rootBinder->bindSingleton(new \Spiral\Core\Config\Proxy(
    AuthInterface::class,
    singleton: true,
    fallbackFactory: static fn() => throw new \LogicException(
        'Unable to receive AuthInterface instance outside of `http` scope.'
    ),
));

// Bind `AuthInterface` in the `http` scope
$container->getBinder('http')
    ->bindSingleton(AuthInterface::class, Auth::class);
```

If a proxy is used outside the `http` scope, the `fallbackFactory` will be called to resolve the dependency.
If the `fallbackFactory` is not provided, a `RecursiveProxyException` will be thrown.
