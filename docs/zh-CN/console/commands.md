# 控制台 — 用户命令

你可以使用启动器向你的应用程序添加新的控制台命令或注册插件命令。该组件默认配置为自动发现位于 `app/src` 目录中的命令。

本页面文档将指导你如何在应用程序中创建和使用控制台命令。无论你是初学者还是经验丰富的开发者，你都能找到入门所需的信息。

## Command 类

要创建一个新命令，你可以扩展 `Symfony\Component\Console\Command\Command` 或 `Spiral\Console\Command`。扩展 `Spiral\Console\Command` 类提供了一些额外的语法糖和便捷方法，可以使构建你的命令更容易。

> **注意**
> 你选择哪个选项取决于你的需求和偏好。这两种方法都是有效的，并且可以用于创建功能齐全的控制台命令。

为了轻松地创建你的第一个命令，请使用脚手架命令：

```terminal
php app.php create:command My
```

> **注意**
> 更多关于脚手架的信息，请阅读 [基础 — 脚手架](../basics/scaffolding.md#console-command) 部分。

执行此命令后，以下输出将确认创建成功：

```output
Declaration of ' [32mMyCommand [39m' has been successfully written into ' [33mapp/src/Endpoint/Console/MyCommand.php [39m'.
```

```php app/src/App/Endpoint/Console/MyCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'my')]
final class MyCommand extends Command
{
    public function __invoke(): int
    {
        // Put your command logic here
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

要调用你的命令，请在控制台中运行以下命令：

```terminal
php app.php my
```

## 特性 (Attributes)

自 3.6 版本发布以来，Spiral 提供了使用 PHP 特性定义控制台命令的能力。这允许使用更直观和简化的方法来定义命令，并清晰地分离关注点。

以下是使用特性定义控制台命令的示例：

```php
namespace App\Api\Cli\Command;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;
use Symfony\Component\Console\Input\InputOption;

#[AsCommand(
    name: 'app:create:user', 
    description: 'Creates a user with the given data')
]
final class CreateUser extends Command
{
    #[Argument]
    private string $email;

    #[Argument(description: 'User password')]
    private string $password;

    #[Argument(name: 'username', description: 'The user name')]
    private string $userName;

    #[Option(shortcut: 'a', name: 'admin', description: 'Set the user as admin')]
    private bool $isAdmin = false;

    public function __invoke(): int
    {
        $user = new User(
            email: $this->email,
            password: $this->password,
        );
        
        $user->setIsAdmin($this->isAdmin);
        
        // Save user to database...

        return self::SUCCESS;
    }
}
```

要定义控制台命令的名称和描述，你可以使用 `Spiral\Console\Attribute\AsCommand` 或 `Symfony\Component\Console\Attribute\AsCommand` 特性。

### 参数 (Arguments)

你可以使用 `Spiral\Console\Attribute\Argument` 特性将类属性标记为控制台命令的参数。

```php
#[Argument]
private string $email;
```

`Argument` 特性没有任何附加参数，表明它将使用属性名称作为参数名称，并且不会包含描述。

```php
#[Argument(description: 'User password')]
private string $password;
```

使用 description 参数可以提供关于参数的简要描述。

```php
#[Argument(name: 'username', description: 'The user name')]
private string $username;
```

name 参数允许你指定一个自定义参数名称，该名称与属性名称不同。

> **注意**
> 任何使用 Argument 特性标记的属性都必须具有标量类型提示，例如 `string`，以指示参数应该接受的值的类型。

如果你想为参数设置默认值，你可以将默认值指定为属性的默认值：

```php
#[Argument]
private string $username = 'guest';
```

在这种情况下，如果在命令调用期间未提供参数的值，则将使用默认值。

在某些情况下，你可能希望在控制台命令中使参数成为可选的。为此，你可以使用属性的 nullable 类型提示，并将默认值指定为 `null`。

```php
#[Argument]
private ?string $username = null;
```

### 选项 (Options)

选项是用户在运行命令时可以指定的附加参数。它们在你的命令类中定义为属性，并使用 `#[Spiral\Console\Attribute\Option]` 特性进行注释。

```php
use Spiral\Console\Attribute\Option;

#[Option(name: 'admin', description: 'Set the user as admin')]
private bool $isAdmin = false;
```

`name` 参数允许你指定一个自定义选项名称，该名称与属性名称不同。

```terminal
php app.php app:create:user --admin
```

你还可以使用 shortcut 参数为该选项指定一个快捷方式：

```php
#[Option(shortcut: 'a', ...)]
private bool $isAdmin = false;
```

该选项可以使用快捷方式 `-a` 而不是输入 `--admin` 来调用。

选项可以定义为必需或可选。要使选项成为必需的，你不需要为属性指定默认值：

```php
#[Option(...)]
private string $status;
```

如果你想使选项成为可选的，你可以将默认值指定为属性的默认值：

```php
#[Option(...)]
private ?string $status = null;
```

使用 PHP 枚举，选项的值将根据枚举值进行验证。

```php
#[Option(description: 'Set the user status')]
private Status $status = Status::new;
```

要接受选项的多个值，你可以使用 `array` 类型提示表示该选项的属性：

```php
#[Option(name: 'role', description: 'Set the user roles')]
private array $role = [];
```

```terminal
php app.php app:create:user --role=foo --role=bar
```

### 提问 (Questions)

默认情况下，当你调用一个未传递必需参数的控制台命令时，应用程序将提示你为每个缺失的参数提供一个值。但是，你可以使用 `Spiral\Console\Attribute\Question` 特性自定义每个参数的提示消息。

它既可以用作属性特性，也可以用作类特性。

```php
#[Option(
    mode: \Symfony\Component\Console\Input\InputOption::VALUE_REQUIRED, 
    ...
)]
private bool $isAdmin = false;
```

当用作类特性时，你需要为每个需要自定义提示的属性定义相应的 `argument` 名称。

```php
#[Question(question: 'Provide user email', argument: 'email')]
final class CreateUserCommand extends Command
{
    #[Argument]
    private string $email;

    // ...
}
```

通过为每个参数设置自定义提示，你可以向用户提供更具体和相关的信息，这可以使交互更直观和简化。 最好使用清晰、简洁且易于理解的提示。

## 签名 (Signature)

`SIGNATURE` 常量提供了另一种定义命令 **名称** 及其 **参数** 和 **选项** 的方法。这可以是一种方便的方式，用于在一个地方指定所有这些信息，而不是单独定义名称、参数和选项。

```php
class SomeCommand extends Command 
{
    protected const SIGNATURE = <<<CMD
        check:http 
            {url : Site url} 
            {--S|skip-ssl-errors : Skip SSL errors}
        CMD;


    public function perform(): int
    {
        $url = $this->argument('url');
        $skipErrors = $this->option('skip-ssl-errors');

        // ...
    }
}
```

在此示例中，`SIGNATURE` 常量定义了一个名为 `check:http` 的命令，该命令具有一个必需参数 `url` 和一个可选选项 `skip-ssl-errors`。

### 参数

你可以在参数名称后包含一个 `?` 字符来使参数成为可选的。例如，`check:http {url?}` 定义了一个可选参数。

你还可以通过在参数名称后包含一个 `=` 字符，然后是默认值来为参数定义默认值。例如，`check:http {url=foo}` 定义了一个具有默认值的参数。

如果你想定义一个期望多个输入值的 **参数**，你可以使用 `[]` 字符。例如，`check:colors {colors[]}` 定义了一个期望多个值的参数。

如果你想使参数成为可选的，你可以在 `[]` 字符后包含一个 `?` 字符，如 `check:colors {colors[]?}`。这将允许用户根据需要省略参数。

### 选项

选项对于指定附加信息或修改命令的行为很有用。它们可以用于启用或禁用某些功能、指定配置文件或其他输入文件，或者设置影响命令行为的其他参数。

选项是用户输入的一种形式，当在命令行上指定时，它以两个连字符 `--` 为前缀。在 `SIGNATURE` 常量中，选项使用语法 `{--name}`、`{--name=value}` 或 `{--n|name}`（如果要使用快捷方式）定义。

你可以对选项使用快捷方式，以便用户在调用命令时更容易地指定选项。

例如，`{--S|skip-ssl-errors}` 定义了一个名为 `skip-ssl-errors` 的选项，其快捷方式为 `S`。这意味着该选项可以在命令行上使用 `--skip-ssl-errors` 或 `--S` 来指定。

使用选项的快捷方式可以使你更容易、更方便地让用户指定他们在调用命令时想要使用的选项。 最好选择清晰简洁的快捷方式名称，这些名称易于记忆和输入。

如果你想定义一个期望多个输入值的 **选项**，你可以使用 `[]` 字符。例如，`check:colors {--colors[]=}` 定义了一个期望多个值的选项。

### 描述

为定义的每个 `argument` 和 `option` 包含一个描述是一个好主意，因为它将帮助用户了解你的命令期望的输入的目的和格式。这可以使你的用户有效地使用你的命令并避免错误。

以下是你可以如何在控制台命令中使用带有描述的参数的示例：

```php
check:http 
    {url : Site url} 
    {--S|skip-ssl-errors : Skip SSL errors}
```

## Perform 方法

你可以将你的用户代码放入 `perform` 方法中。`perform` 方法支持方法注入，并提供 `$this->input` 和 `$this->output` 属性来处理用户输入。

```php
protected function perform(MyService $service): int
{
    $this->output->writeln($service->doSomething());
    
    return self::SUCCESS;
}
```

要通过参数和/或选项获取用户数据，你可以使用 `$this->argument("argName")` 或 `$this->option("optName")`。

> **注意**
> 此外，[这里](/cookbook/console-validation.md) 你可以找到有关如何将 `spiral/filters` 组件用于控制台命令的信息。

## 辅助方法 (Helper Methods)

你可以在 `Spiral\Console\Command` 中使用一组辅助方法。给出的示例旨在在 `perform` 方法中调用。

将内容写入输出：

```php
$this->writeln('hello world');
```

将内容写入输出，而不转到新行：

```php
$this->write('hello world');
```

将格式化的输出写入输出，而不转到新行：

```php
$this->sprintf('Hello, <comment>%s</comment>', $name);
```

> **注意**
> 此方法与 `sprintf` 定义兼容。

检查当前的详细模式是否高于 `OutputInterface::VERBOSITY_VERBOSE`：

```php
dump($this->isVerbose());
```

> **注意**
> 你可以在控制台命令中自由使用 `dump` 方法。

渲染表格：

```php
$table = $this->table([
    'Column #1:',
    'Column #2:',
]);

foreach ($data as $row)
{
    $table->addRow([
        $row[1],
        $row[2]
    ]);
}

$table->render();
```

添加表格分隔符：

```php
$table->addRow(new Symfony\Component\Console\Helper\TableSeparator());
```

确定输入选项是否存在：

```php
$this->hasOption(...);
```

确定输入参数是否存在：

```php
$this->hasArgument(...);
```

请求确认：

```php
$status = $this->confirm('Are you sure?', default: false);
```

提问：

```php
$status = $this->ask('Are you sure?', default: 'no');
```

提出多项选择题：

```php
$name = $this->choiceQuestion(
    'Which of the following is package manager?',
    ['composer', 'django', 'phoenix', 'maven', 'symfony'],
    default: 2,
    allowMultipleSelections: true
);
```

提示用户输入，但从控制台中隐藏答案：

```php
$status = $this->secret('User password');
```

将消息作为信息输出写入：

```php
$this->info('Some message');
```

将消息作为注释输出写入：

```php
$this->comment('Some message');
```

将消息作为问题输出写入：

```php
$this->question('Some question');
```

将消息作为错误输出写入：

```php
$this->error('Some error');
```

将消息作为警告输出写入：

```php
$this->warning('Some warning');
```

将消息作为警报输出写入：

```php
$this->alert('Some alert');
```

写一个空白行：

```php
$this->newLine();
$this->newLine(count: 5);
```

## ApplicationInProduction

该组件提供的 `Spiral\Console\Confirmation\ApplicationInProduction` 类使在应用程序以生产模式运行时，在运行命令之前轻松地要求用户进行确认。这可以帮助防止对生产环境进行意外或无意的更改。

要使用它，你可以将它的一个实例注入到你的命令的 `perform()` 方法中。然后，你可以使用 `confirmToProceed()` 方法在继续执行命令之前要求用户进行确认。

```php
use Spiral\Console\Confirmation\ApplicationInProduction;

final class MigrateCommand extends Command
{
    protected const NAME = 'db:migrate';

    public function perform(ApplicationInProduction $confirmation): int
    {
        if (!$confirmation->confirmToProceed()) {
            return self::FAILURE;
        }
        
        // run migrations...
    }
}
```

## 参数和选项

Command 类使你可以轻松地为控制台命令定义参数和选项。通过设置 `ARGUMENTS` 和 `OPTIONS` 常量：

```php
const ARGUMENTS = [
    ['argName', InputArgument::REQUIRED, 'Argument name.']
];
    
const OPTIONS = [
    ['optName', 'c', InputOption::VALUE_NONE, 'Some option.']
];
```

## 事件

| 事件                                | 描述                                                     |
|--------------------------------------|-----------------------------------------------------------------|
| Spiral\Console\Event\CommandStarting | 该事件将在执行控制台命令`之前`触发。 |
| Spiral\Console\Event\CommandFinished | 该事件将在执行控制台命令`之后`触发。  |

> **注意**
> 要了解有关调度事件的更多信息，请参阅我们文档中的 [事件](../advanced/events.md) 部分。
