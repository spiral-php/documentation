# 验证 - 快速入门

Spiral 的验证组件允许你验证用户提交的或从外部来源接收到的数据。

该组件开箱即不包含任何验证器实现。相反，它提供了一组接口和抽象类，定义了验证器的预期行为。

## 安装

要启用该组件，你只需将 `Spiral\Validation\Bootloader\ValidationBootloader` 添加到引导程序列表中。

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Validation\Bootloader\ValidationBootloader::class,
        // ...
    ];
}
```

在 [框架 - 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的内容。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Validation\Bootloader\ValidationBootloader::class,
    // ...
];
```

在 [框架 - 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的内容。
:::

::::

## 配置

有三个验证器桥可用于 Spiral 验证组件：

-   [Spiral Validator](./spiral.md) - 这是默认的验证器桥。它是一个简单、轻量级的验证器，可以处理基本的验证任务。
-   [Symfony Validator](./symfony.md) - 此验证器桥提供与 Symfony Validator 组件的集成，这是一个更强大且功能丰富的验证库。
-   [Laravel Validator](./laravel.md) - 此验证器桥提供与 Laravel Validator 的集成，Laravel Validator 是在 Laravel 框架中使用的验证组件。

你可以根据自己的需求和偏好，在你的应用程序中使用任何这些验证器桥。

大多数应用程序使用单一的验证器实现，但 Spiral 允许你根据需要，在应用程序中使用多个验证器。在这种情况下，你可以在 `app/config/validation.php` 配置文件中定义一个默认验证器。

```php app/config/validation.php
return [
    'defaultValidator' => 'my-validator',
    // ...
];
```

除了在配置文件中设置默认验证器之外，你还可以使用 `Spiral\Validation\Bootloader\ValidationBootloader` 设置默认验证器。

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Validation\Bootloader\ValidationBootloader;

final class AppBootloader extends Bootloader
{
    public function boot(ValidationBootloader $validation): void
    {
        $validation->setDefaultValidator('my-validator');
    }
}
```

## 使用

在本节中，我们将向你展示如何使用验证组件来验证数据。

这是一个使用默认验证器验证数据的示例：

```php app/src/Interface/Controller/UserController.php
use Spiral\Http\Request\InputManager;
use Spiral\Validation\ValidatorInterface;

class UserController
{
    public function create(InputManager $input, ValidatorInterface $validator)
    {
        $validator = $validator->validate([
            'username' => $input->post('username'),
            'email' => $input->post('email'),
        ], [
            'username' => 'required',
            'email' => 'required|email',
        ]);
        
        if (!$validator->isValid()) {
            $errors = $validator->getErrors();
            // ...
        }

        // Store the user in the database...
    }
}
```

`Spiral\Validation\ValidationInterface` 具有一个 `validate` 方法，该方法接受验证数据、验证规则和上下文。该方法返回一个 `Spiral\Validation\ValidatorInterface` 实例。

当应用程序从容器中解析 `Spiral\Validation\ValidatorInterface` 时，它将从 `Spiral\Validation\ValidationProviderInterface` 接口请求它，该接口负责提供验证器实例。`ValidationProviderInterface` 根据已设置的默认验证器（在配置文件中或使用 `ValidationBootloader`）确定要使用的验证器。

如果你想使用不同的验证器，你可以从 `ValidationProviderInterface` 请求它。

```php app/src/Interface/Controller/UserController.php
use Spiral\Http\Request\InputManager;
use Spiral\Validation\ValidationProviderInterface;

class UserController
{
    public function create(InputManager $input, ValidationProviderInterface $provider)
    {
        $validator = $provider->getValidation('my-validator')->validate([
            'username' => $input->post('username'),
            'email' => $input->post('email'),
        ], [
            'username' => 'required',
            'email' => 'required|email',
        ]);

        // Validate the data...
        // Store the user in the database...
    }
}
```

## 自定义验证器

要创建自定义验证器，你需要创建一个实现 `ValidationInterface` 接口的类。此接口定义了一个方法 `validate`，该方法接受一个要验证的数据数组和一个验证规则数组作为参数，并返回一个验证器对象。

```php
namespace App\Validator;

use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidatorInterface;

final class MyValidation implements ValidationInterface
{
    public function validate(mixed $data, array $rules, $context = null): ValidatorInterface
    {
        return (new MyValidator(
            new MyValidationService($rules)
        ))
          ->withData($data)
          ->withContext($context);
    }
}
```

你还需要创建一个实现 `ValidatorInterface` 的验证器对象，并负责执行实际的验证。这可以是一个独立的类，也可以是一个扩展 Spiral 或另一个库提供的验证器类的类。验证器应该实现将数据与验证规则进行比较并返回一个布尔值的逻辑，该布尔值指示数据是否有效。

```php
namespace App\Validator;

use Spiral\Validation\ValidatorInterface;

final class MyValidator implements ValidatorInterface
{
    protected array|object $data = [];
    protected mixed $context = null;
        
    public function __construct(
        private readonly MyValidationService $validationService
    ) {}
    
    public function isValid(): bool
    {
        return $this->validationService->validate($this->data, $this->context);
    }
    
    public function getErrors(): array
    {
        return $this->validationService->getErrors();
    }
    
    // other required methods
}
```

现在我们可以注册创建的验证器。为此，使用 `Spiral\Validation\ValidationProvider` 类的 `register` 方法。

```php app/src/Application/Bootloader/ValidatorBootloader.php
namespace App\Application\Bootloader;

use App\Validation;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Validation\Bootloader\ValidationBootloader;
use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidationProvider;

final class ValidatorBootloader extends Bootloader
{
    public function boot(ValidationProvider $provider): void
    {
        $provider->register(
            'my-validator',
            static fn(Validation $validation): ValidationInterface => new MyValidation()
        );
    }
}
```

> **另请参阅**
> 在 [框架 - 引导程序](../framework/bootloaders.md) 部分阅读更多关于引导程序的内容。

值得注意的是，这只是一个你如何在 Spiral 中创建自定义验证器的例子。还有许多其他方法和技术，你可以使用它们来根据你的特定需求和要求自定义和扩展验证过程。
