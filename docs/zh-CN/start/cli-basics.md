# 开始使用 - 第一个 CLI 命令

Spiral 提供了一种便捷的方法来创建控制台应用程序。它内置支持控制台命令，使您能够为您的应用程序开发命令行界面（CLI）。控制台命令使您能够自动化任务，执行维护操作，并以超越标准 Web 界面限制的方式与您的应用程序进行交互。

在 Spiral 中使用控制台命令非常简单。该框架提供了一个用户友好的界面，利用了 `symfony/console` 包的强大功能。

让我们逐步创建第一个控制台命令。

## 创建一个命令

为了轻松地创建您的第一个命令，请使用脚手架命令：

```terminal
php app.php create:command CurrentDate
```

> **注意**
> 在 [基础知识 — 脚手架](../basics/scaffolding.md#console-command) 部分阅读有关脚手架的更多信息。

执行此命令后，以下输出将确认创建成功：

```output
Declaration of ' [32mCurrentDateCommand [39m' has been successfully written into ' [33mapp/src/Endpoint/Console/CurrentDateCommand.php [39m'.
```

现在，让我们将一些逻辑注入到我们新创建的命令中。

这是一个将当前日期输出到控制台的控制台命令示例：

```php app/src/App/Endpoint/Console/CurrentDateCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'current:date')]
final class CurrentDateCommand extends Command
{
    #[Argument(description: 'Date format')]
    public string $format = 'Y-m-d';

    public function __invoke(): int
    {
        $this->writeln(\date($this->format));

        return self::SUCCESS;
    }
}
```

默认情况下，Spiral 配置为通过 [静态分析组件](../advanced/tokenizer.md) 自动发现位于 `app/src` 目录中的命令。这意味着您不必手动注册命令或为它们创建单独的配置文件。

## 运行命令

要检索您命令的帮助信息，请在您的终端中执行以下命令：

```terminal
php app.php help current:date
```

这将显示命令的签名、描述以及任何可用的参数或选项。

```output
 [33mDescription: [39m
  Get current date

 [33mUsage: [39m
  current:date [<format>]

 [33mArguments: [39m
   [32mformat [39m                Date format [33m [default: "Y-m-d"] [39m

 [33mOptions: [39m
   [32m-h, --help [39m            Display help for the given command. When no command is given display help for the  [32mlist [39m command
   [32m-q, --quiet [39m           Do not output any message
   [32m-V, --version [39m         Display this application version
   [32m    --ansi|--no-ansi [39m  Force (or disable --no-ansi) ANSI output
   [32m-n, --no-interaction [39m  Do not ask any interactive question
   [32m-v|vv|vvv, --verbose [39m  Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

<br>

**就是这样！您已成功在 Spiral 中设置了您的第一个控制台命令。**

<hr>

## 接下来是什么？

现在，通过阅读一些文章来深入了解基础知识：

*   [Cli 配置](../console/configuration.md)
*   [创建命令](../console/commands.md)
*   [拦截器](../console/interceptors.md)
*   [命令输入验证](../cookbook/console-validation.md)
*   [脚手架](../basics/scaffolding.md)
