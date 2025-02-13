# Cookbook — 控制台命令输入验证

`spiral/filters` 是一个强大的组件，用于过滤和验证输入数据。它允许您创建过滤器，这些过滤器可用于映射和验证来自各种来源的输入数据，例如 HTTP 请求、gRPC 请求和控制台命令。

通过过滤器，您可以将控制台命令输入数据映射到结构化的对象，然后使用这些对象来验证输入数据。

使用过滤器验证控制台命令输入数据可以帮助确保您的命令接收到干净、有效的数据，并且可以更容易地管理您的验证逻辑。此外，过滤器可以在不同的控制台命令之间复用，这有助于减少代码重复并使维护应用程序更容易。

## 工作原理

![Console Filters](https://user-images.githubusercontent.com/773481/211006701-170ee6b7-158c-4310-96bc-d675ff22aac2.png)
*控制台应用程序中过滤和验证输入数据的过程图示*

## 控制台命令

让我们想象一下，我们有一个控制台命令，该命令接受用户的姓名和电子邮件地址作为输入：

```php
namespace App\Command;

use Spiral\Console\Command;

final class UserRegister extends Command
{
    protected const SIGNATURE = <<<CMD
        user:register
        {username : 用户名}
        {email : 用户邮箱地址}
        {--a|admin : 标记为管理员}
        {--s|send-verification-email : 向用户发送验证邮件}
CMD;

    public function perform(): int
    {
        // ...

        return self::SUCCESS;
    }
}
```

并且我们希望在使用输入数据创建新用户之前验证它。我们可以通过使用过滤器来实现这一点。

## 输入源

为了将过滤器组件与控制台命令一起使用，您需要将一个 `Spiral\Filters\InputInterface` 的实例绑定到您的控制台命令的输入。

以下是一个如何将 `InputInterface` 的实例绑定到您的控制台命令的输入的示例：

```php
namespace App\Application\Console;

use Spiral\Filters\InputInterface as FilterInputInterface;
use Symfony\Component\Console\Input\InputInterface;

final class ConsoleInput implements FilterInputInterface
{
    public function __construct(
        private readonly InputInterface $input
    ) {
    }

    public function withPrefix(string $prefix, bool $add = true): self
    {
        return $this;
    }

    public function getValue(string $source, string $name = null): mixed
    {
        return match ($source) {
            'argument' => $this->input->getArgument($name),
            'option' => $this->input->getOption($name),
            default => throw new \InvalidArgumentException('Invalid input source'),
        };
    }

    public function hasValue(string $source, string $name): bool
    {
        return match ($source) {
            'argument' => $this->input->hasArgument($name),
            'option' => $this->input->hasOption($name),
            default => throw new \InvalidArgumentException('Invalid input source'),
        };
    }
}
```

为了将您的 `ConsoleInput` 实现与您的控制台命令一起使用，您需要在启动器中注册它。

这里有一个例子：

```php
namespace App\Application\Bootloader;

use App\Application\Console\ConsoleInput;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Filters\InputInterface;

final class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        InputInterface::class => ConsoleInput::class,
    ];
}
```

## Attributes (属性)

`spiral/filters` 组件使用属性来定义应用于验证输入数据的规则。

在我们的应用程序中，有两种类型的属性可用于从控制台输入请求数据：

### Argument (参数)

该属性表示作为 `argument` 传递给控制台命令的输入字段。

```php
namespace App\Application\Console\Attribute;

use Spiral\Attributes\NamedArgumentConstructor;
use Spiral\Filters\Attribute\Input\AbstractInput;
use Spiral\Filters\InputInterface;

#[\Attribute(\Attribute::TARGET_PROPERTY), NamedArgumentConstructor]
final class Argument extends AbstractInput
{
    public function __construct(
        public readonly ?string $key = null,
    ) {
    }

    public function getValue(InputInterface $input, \ReflectionProperty $property): mixed
    {
        return $input->getValue('argument', $this->getKey($property));
    }

    public function getSchema(\ReflectionProperty $property): string
    {
        return 'argument:' . $this->getKey($property);
    }
}
```

### Option (选项)

该属性表示作为 `option` 传递给控制台命令的输入字段。

```php
namespace App\Application\Console\Attribute;

use Spiral\Attributes\NamedArgumentConstructor;
use Spiral\Filters\Attribute\Input\AbstractInput;
use Spiral\Filters\InputInterface;

#[\Attribute(\Attribute::TARGET_PROPERTY), NamedArgumentConstructor]
final class Option extends AbstractInput
{
    public function __construct(
        public readonly ?string $key = null,
    ) {
    }

    public function getValue(InputInterface $input, \ReflectionProperty $property): mixed
    {
        return $input->getValue('option', $this->getKey($property));
    }

    public function getSchema(\ReflectionProperty $property): string
    {
        return 'option:' . $this->getKey($property);
    }
}
```

## 创建过滤器

现在我们已经定义了将用于从控制台输入请求数据的属性，我们可以创建一个过滤器：

```php
namespace App\Command;

use App\Application\Console\Attribute\Argument;
use App\Application\Console\Attribute\Option;
use Spiral\Filters\Model\Filter;

class UserRegisterFilter extends Filter
{
    #[Argument]
    public string $username;

    #[Argument]
    public string $email;

    #[Option]
    public bool $admin = false;

    #[Option(key: 'send-verification-email')]
    public bool $sendVerificationEmail = false;
}
```

## 使用过滤器

现在我们已经创建了一个过滤器，我们可以使用它来验证传递给我们的控制台命令的输入数据：

```php
namespace App\Command;

use Spiral\Console\Command;

final class UserRegister extends Command
{
    protected const SIGNATURE = <<<CMD
        user:register
        {username : 用户名}
        {email : 用户邮箱地址}
        {--a|admin : 标记为管理员}
        {--s|send-verification-email : 向用户发送验证邮件}
CMD;

    public function perform(UserRegisterFilter $input): int
    {
        $this->writeln(\sprintf('Username: %s', $input->username));
        $this->writeln(\sprintf('Email: %s', $input->email));
        $this->writeln(\sprintf('Is admin: %s', $input->admin ? 'yes' : 'no'));

        // $user = new User(
        //     username: $filter->username,
        //     email: $filter->email,
        //    admin: $filter->admin
        // );

        // 将用户存储在数据库中...

        if ($input->sendVerificationEmail) {
            $this->writeln('发送验证电子邮件...');

            // 发送验证电子邮件...
        }

        $this->writeln(\sprintf('用户 %s 已注册', $input->username));

        return self::SUCCESS;
    }
}
```

现在您可以运行控制台命令：

```terminal
php app.php user:register john_smith john@site.com -as
```

## 验证

`spiral/filters` 组件使用 `spiral/validation` 组件来验证输入数据。

> **注意**
> 该组件依赖于 [Validation](../validation/factory.md) 组件，请务必先阅读它。

为了使用验证，您需要定义应用于验证输入数据的规则。

这里有一个例子：

```php
use App\Application\Console\Attribute\Argument;
use App\Application\Console\Attribute\Option;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class UserRegisterFilter extends Filter implements HasFilterDefinition
{
    #[Argument]
    public string $username;

    #[Argument]
    public string $email;

    #[Option]
    public bool $admin = false;

    #[Option(key: 'send-verification-email')]
    public bool $sendVerificationEmail = false;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => ['required', 'string', ['string::longer', 3], ['string::shorter', 32]],
            'email' => ['required', 'email'],
        ]);
    }
}
```

现在，如果使用无效的输入数据运行控制台命令，您将收到一条错误消息：

```terminal
php app.php user:register jh john
```

类似于这样：

```output
[Spiral\Filters\Exception\ValidationException]
给定的数据无效。
```

但是，错误消息将不包含有关验证错误的详细信息。如果您想获取详细信息，您将需要为控制台命令创建一个拦截器，该拦截器将处理 `Spiral\Filters\Exception\ValidationException` 异常并显示有关验证错误的详细信息：

```php
namespace App\Command;

use Spiral\Console\Command;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Symfony\Component\Console\Output\OutputInterface;

class HandleValidationExceptions implements CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
            return $core->callAction($controller, $action, $parameters);
        } catch (\Spiral\Filters\Exception\ValidationException $e) {
            $output = $parameters['output'];
            \assert($output instanceof OutputInterface);

            $output->writeln('<fg=red>验证错误：</>');
            foreach ($e->errors as $key => $error) {
                $output->writeln(\sprintf('<fg=red>%s: %s</>', $key, $error));
            }

            return Command::FAILURE;
        }
    }
}
```

并在 `app\config\console.php` 中注册拦截器：

```php app\config\console.php
return [
    'interceptors' => [
        App\Command\HandleValidationExceptions::class,
        // ...
    ],
];
```

> **另请参阅**
> 在 [Console — Interceptors](../console/interceptors.md) 部分阅读有关控制台拦截器的更多信息。

就是这样。现在，如果您使用无效的输入数据运行控制台命令：

```terminal
php app.php user:register jh john -as
```

您将收到一条错误消息，其中包含有关验证错误的详细信息，如下所示：

```output
验证错误：
username: 文本必须大于或等于 3 个字符。
email: 必须是有效的电子邮件地址。
```
