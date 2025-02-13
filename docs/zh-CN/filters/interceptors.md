# 过滤器 - 拦截器

Spiral 提供了一种供开发者通过使用拦截器来定制其过滤器行为的方式。拦截器是一段代码，在过滤器创建之后执行，它允许开发者钩入过滤器处理管道来执行某些操作。

拦截器可以用于多种用途，例如处理错误或向过滤器添加额外的上下文。

要使用拦截器，您需要实现 `Spiral\Core\CoreInterceptorInterface` 接口。

## 过滤器拦截器

让我们设想我们需要检查用户授权以访问某个过滤器。我们可以使用拦截器来实现：

```php app/src/Endpoint/Web/Filter/StoreUser.php
namespace App\Endpoint\Web\Filter;

use Spiral\Domain\Annotation\Guarded;

#[Guarded(permission: 'user.store')]
final class StoreUser extends Filter
{
    #[Post]
    public string $username;
    
    #[Post]
    public string $email;
    
    //...
}
```

> **另见**
> 在 [安全 — 基于角色的访问控制](../security/rbac.md) 章节中阅读更多关于用户基于角色的访问控制的信息。

这是一个检查授权的简单拦截器的例子：

```php
namespace App\Filter;

use Spiral\Attributes\ReaderInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Domain\Annotation\Guarded;
use Spiral\Http\Exception\ClientException\UnauthorizedException;
use Spiral\Security\GuardInterface;

class CheckGuardInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly GuardInterface $guard,
        private readonly ReaderInterface $reader,
    ) {
    }

    public function process(string $filterClass, string $action, array $parameters, CoreInterface $core): mixed
    {
        $refClass = new \ReflectionClass($filterClass);
        // 从过滤器类中读取属性并检查它是否具有 Guarded 注解
        $guarded = $this->reader->firstClassMetadata($refClass, Guarded::class);

        // 如果过滤器具有 Guarded 属性，请检查用户是否具有权限
        if ($guarded && !$this->guard->allows($guarded->permission ?: $filterClass)) {
            throw new UnauthorizedException($guarded->errorMessage ?: 'Access denied');
        }

        // 如果过滤器没有 Guarded 属性，或者用户具有权限，则继续
        return $core->callAction($filterClass, $action, $parameters);
    }
}
```

要使用此拦截器，您需要在配置文件 `app/config/filters.php` 中注册它。

```php
use App\Filter\CheckGuardInterceptor;
use Spiral\Filters\Model\Interceptor\PopulateDataFromEntityInterceptor;
use Spiral\Filters\Model\Interceptor\ValidateFilterInterceptor;

return [    
     'interceptors' => [
        CheckGuardInterceptor::class,
        PopulateDataFromEntityInterceptor::class,
        ValidateFilterInterceptor::class,
     ],
     // ...
];
```

> **注意**
> 请注意，我们在列表中有 `PopulateDataFromEntityInterceptor` 和 `ValidateFilterInterceptor`，它们是过滤器正常工作所必需的。
