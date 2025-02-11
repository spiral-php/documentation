# Temporal — 使用

在 Temporal 中，工作流是一个长时间运行的进程，由一系列相互关联的活动组成。工作流允许开发者定义和执行可能跨越多个服务甚至系统的复杂进程。

工作流由 Temporal 工作流引擎执行。工作流可以由外部事件触发（例如用户请求或消息队列中的消息），或者可以被安排在特定时间间隔运行。

让我们看一个简单的示例，该工作流每 5 分钟 ping 一次网站，如果网站宕机则发送通知。

## 工作流定义

工作流是一个 PHP 类，其中包含一个使用 `#[WorkflowMethod]` 属性注释的单一方法。此方法将用作启动工作流的入口点。

> **阅读更多**
> 有关工作流的更多信息，请参阅 [Temporal 文档](https://docs.temporal.io/workflows)。

要创建工作流，请运行以下命令：

```terminal
php app.php create:workflow WebsiteStatus
```

此命令将在 `app/src/Endpoint/Temporal/Workflow` 目录中创建一个新的工作流类，其内容如下：

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Temporal\Workflow;

use Temporal\Workflow\WorkflowInterface;
use Temporal\Workflow\WorkflowMethod;

#[WorkflowInterface]
class WebsiteStatusWorkflow
{
    #[WorkflowMethod]
    public function handle()
    {
        // TODO: Implement handle method
    }
}
```

让我们向工作流添加一些逻辑：

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
namespace App\Endpoint\Temporal\Workflow;

use Carbon\CarbonInterval;
use Temporal\Workflow\WorkflowInterface;
use Temporal\Workflow\WorkflowMethod;
use Temporal\Workflow;

#[WorkflowInterface]
class WebsiteStatusWorkflow
{
    private bool $isDownNotified = false;
    private bool $isRecoveryNotified = false;
    private int $downTime = 0;

    #[WorkflowMethod]
    public function handle(string $url, int $intervalInMinutes = 5)
    {
        while (true) {
            // 这里我们将 ping 网站并获取状态
            $status = ...

            if ($status === false) {
                // 仅在网站宕机时发送一次通知
                if (!$this->isDownNotified) {
                    // 这里我们将发送关于停机的通知
                }

                $this->isDownNotified = true;
                // 增加停机时间 5 分钟
                $this->downTime += $intervalInMinutes;
            } else {
                // 仅在网站恢复时发送一次通知
                if (!$this->isRecoveryNotified) {
                    // 这里我们将发送关于恢复的通知，包含总停机时间
                }

                $this->downTime = 0;
                $this->isRecoveryNotified = true;
            }

            // 等待 5 分钟
            yield Workflow::timer(CarbonInterval::minutes($intervalInMinutes));
        }
    }
}
```

正如您所看到的，我们的工作流是一个简单的循环，它将每 5 分钟 ping 一次网站，并在网站宕机和恢复时发送通知。我们还跟踪总停机时间，并在网站恢复时将其发送在通知中。

为了 ping 网站和发送通知，我们将使用活动。让我们创建它们。

> **注意**
> 当您运行应用程序时，工作流类将自动在 Temporal 服务器中注册。Spiral 将查找所有具有 `Temporal\Workflow\WorkflowInterface` 属性的类，并在 Temporal 服务器中注册它们。

> **警告**
> 您不能在工作流中使用 DI、IO 操作或任何其他阻塞操作。如果需要使用任何这些操作，则需要使用活动。

## 活动定义

活动是工作流的构建块。活动执行单个、定义明确的操作（短时间或长时间运行），例如调用另一个服务、转码媒体文件、发送电子邮件消息等。

> **阅读更多**
> 有关活动的更多信息，请参阅 [Temporal 文档](https://docs.temporal.io/activities)。

让我们创建一个 ping 网站的活动：

```terminal
php app.php create:activity PingWebsite --method=ping:bool
```

此命令将在 `app/src/Endpoint/Temporal/Activity` 目录中创建一个新的活动类，其内容如下：

```php app/src/Endpoint/Temporal/Activity/PingWebsiteActivity.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Temporal\Activity;

use React\Promise\PromiseInterface;
use Temporal\Activity\ActivityInterface;
use Temporal\Activity\ActivityMethod;

#[ActivityInterface]
class PingWebsiteActivity
{
    /**
     * @return PromiseInterface<bool>
     */
    #[ActivityMethod(name: 'ping')]
    public function ping(): bool
    {
        // TODO: Implement activity method
    }
}
```

> **注意**
> 当您运行应用程序时，活动类将自动在 Temporal 服务器中注册。Spiral 将查找所有具有 `Temporal\Activity\ActivityInterface` 属性的类，并在 Temporal 服务器中注册它们。

让我们向活动添加一些逻辑：

```php app/src/Endpoint/Temporal/Activity/PingWebsiteActivity.php
namespace App\Endpoint\Temporal\Activity;

use React\Promise\PromiseInterface;
use Temporal\Activity\ActivityInterface;
use Temporal\Activity\ActivityMethod;

#[ActivityInterface]
class PingWebsiteActivity
{
    /**
     * @return PromiseInterface<bool>
     */
    #[ActivityMethod(name: 'ping')]
    public function ping(string $domain): bool
    {
        // 这里我们将 ping 网站并获取状态
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $domain);
        curl_setopt($ch, CURLOPT_HEADER, TRUE);
        curl_setopt($ch, CURLOPT_NOBODY, TRUE); // 移除主体
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
        $head = curl_exec($ch);
        $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);

        return $httpCode === 200;
    }
}
```

要在工作流中使用活动，我们需要在工作流构造函数中初始化它：

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
use Temporal\Internal\Workflow\ActivityProxy;
use Temporal\Activity\ActivityOptions;
use App\Endpoint\Temporal\Activity\PingWebsiteActivity;

#[WorkflowInterface]
class WebsiteStatusWorkflow
{
    // ...
    
    private PingWebsiteActivity|ActivityProxy $pingActivity;

    public function __construct()
    {
        $this->pingActivity = Workflow::newActivityStub(
            PingWebsiteActivity::class,
            ActivityOptions::new()
                ->withStartToCloseTimeout(5)
        );
    }
    
    //...
}
```

`Workflow::newActivityStub` 方法将创建一个代理类 `Temporal\Internal\Workflow\ActivityProxy`，该类将用于调用该活动。尽管该活动是一个 PHP 类，但每个活动的方**法调用都将被发送到 Temporal 服务器，并且服务器将通过其名称执行实际的活动。

现在我们可以在工作流中使用该活动：

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
// ...
    #[WorkflowMethod]
    public function handle(string $url, int $intervalInMinutes = 5)
    {
        while (true) {
            // 这里我们将 ping 网站并获取状态
            $status = yield $this->pingActivity->ping($url);

            if ($status === false) {

// ...
```

当您调用活动方法时，将返回一个 Promise 对象。当活动完成时，该 Promise 将被解析。我们使用 `yield` 来等待 Promise 被解析并将返回活动的结果。

```php
yield $this->pingActivity->ping($url)
```

活动调用同步阻塞，直到活动完成、失败或超时。即使活动执行需要几个月的时间，工作流代码仍然将其视为一个单一的同步调用。

在某些情况下，您可能希望并行执行多个活动。例如，您可能希望调用服务以 ping 您的站点，例如：

```php
$status = yield $this->pingActivity->pingFromEurope($url);
$status = yield $this->pingActivity->pingFromAsia($url);
$status = yield $this->pingActivity->pingFromAmerica($url);
```

如果使用 `yield` 来调用活动，它们将按顺序执行。要并行执行活动，您需要使用 `\Temporal\Promise\Promise::all` 方法：

```php
use Temporal\Promise\Promise;

[$statusEurope, $statusAsia, $statusAmerica] = yield Promise::all([
    $this->pingActivity->pingFromEurope($url),
    $this->pingActivity->pingFromAsia($url),
    $this->pingActivity->pingFromAmerica($url),
]);
```

在这种情况下，所有活动将并行执行，结果将作为一个数组返回。

### 发送通知

为了发送通知，我们将使用另一个活动：

```terminal
php app.php create:activity SendNotification --method=sendFailedNotification:void --method=sendRecoveryNotification:void
```

此命令将在 `app/src/Endpoint/Temporal/Activity` 目录中创建一个新的活动类，其内容如下：

```php app/src/Endpoint/Temporal/Activity/SendNotificationActivity.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Temporal\Activity;

use React\Promise\PromiseInterface;
use Temporal\Activity\ActivityInterface;
use Temporal\Activity\ActivityMethod;

#[ActivityInterface]
class SendNotificationActivity
{
    /**
     * @return PromiseInterface<void>
     */
    #[ActivityMethod(name: 'sendFailedNotification')]
    public function sendFailedNotification(): void
    {
        // TODO: Implement activity method
    }
    
    /**
     * @return PromiseInterface<void>
     */
    #[ActivityMethod(name: 'sendRecoveryNotification')]
    public function sendRecoveryNotification(): void
    {
        // TODO: Implement activity method
    }
}
```

让我们向活动添加一些逻辑：

```php app/src/Endpoint/Temporal/Activity/SendNotificationActivity.php
namespace App\Endpoint\Temporal\Activity;

use React\Promise\PromiseInterface;
use Spiral\Mailer\MailerInterface;
use Temporal\Activity\ActivityInterface;
use Temporal\Activity\ActivityMethod;

#[ActivityInterface]
class SendNotificationActivity
{
    public function __construct(
        private readonly MailerInterface $mailer,
    ) {
    }

    /** @return PromiseInterface<void> */
    #[ActivityMethod(name: 'sendFailedNotification')]
    public function sendFailedNotification(string $domain): void
    {
        $text = "Website {$domain} is down.";

        // $this->mailer->send(...);
    }

    /** @return PromiseInterface<void> */
    #[ActivityMethod(name: 'sendRecoveryNotification')]
    public function sendRecoveryNotification(string $domain, int $downTime): void
    {
        $text = "Website {$domain} is up after {$downTime} minutes of downtime";

        // $this->mailer->send(...);
    }
}
```

我们还可以为活动指定任务队列。任务队列是活动的逻辑分组。默认情况下，所有工作流和活动都分配给 `default` 任务队列。您可以使用 PHP 属性为活动指定任务队列：

```php
use Spiral\TemporalBridge\Attribute\AssignWorker;

#[AssignWorker('mailer')]
#[ActivityInterface]
class SendNotificationActivity
{
}
```

然后，我们可以告诉工作流为该活动使用此任务队列：

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
$this->mailActivity = Workflow::newActivityStub(
    SendNotificationActivity::class,
    ActivityOptions::new()
        ->withStartToCloseTimeout(5)
        ->withTaskQueue('mailer')
);
```

要在工作流中使用该活动，我们需要在工作流构造函数中初始化它：

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
use Temporal\Internal\Workflow\ActivityProxy;
use Temporal\Activity\ActivityOptions;
use App\Endpoint\Temporal\Activity\SendNotificationActivity;

#[WorkflowInterface]
class WebsiteStatusWorkflow
{
    // ...
    
    private SendNotificationActivity|ActivityProxy $mailActivity;

    public function __construct()
    {
        // ...
        
        $this->mailActivity = Workflow::newActivityStub(
            SendNotificationActivity::class,
            ActivityOptions::new()
                ->withStartToCloseTimeout(5)
                ->withTaskQueue('mailer')
        );
    }
    
    //...
}
```

现在我们可以在工作流中使用该活动：

```php app/src/Endpoint/Temporal/Workflow/WebsiteStatusWorkflow.php
if ($status === false) {
    // 仅在网站宕机时发送一次通知
    if (!$this->isDownNotified) {
        yield $this->mailActivity->sendFailedNotification($url);
    }

    $this->isDownNotified = true;
    // 增加停机时间
    $this->downTime += $intervalInMinutes;
} else {
    // 仅在网站恢复时发送一次通知
    if (!$this->isRecoveryNotified) {
        yield $this->mailActivity->sendRecoveryNotification($url, $this->downTime);
    }

    $this->downTime = 0;
    $this->isRecoveryNotified = true;
}
```

就这样。现在我们可以启动工作流了。

## 启动工作流

### Temporal 开发服务器

在运行工作流之前，我们需要启动 Temporal 服务器。

要启动服务器，请运行以下命令：

```terminal
temporal server start-dev
```

### RoadRunner 服务器

为了运行应用程序，我们需要启动预先配置了 Temporal 插件的 RoadRunner 服务器：

```yaml .rr.yaml
version: '3'
rpc:
  listen: 'tcp://127.0.0.1:6001'

server:
  command: 'php app.php'
  relay: pipes

temporal:
  address: localhost:7233
  activities:
    num_workers: 10
```

然后运行以下命令：

```terminal
./rr serve
```

### 运行工作流

我们可以使用 `Temporal\Client\WorkflowClientInterface` 接口运行工作流。让我们创建一个控制台命令来启动工作流：

```terminal
php app.php create:command CheckStatus
```

此命令将在 `app/src/Endpoint/Console` 目录中创建一个新的控制台命令类，其内容如下：

```php app/src/Endpoint/Console/CheckStatusCommand.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'check:status')]
final class CheckStatusCommand extends Command
{
    public function __invoke(): int
    {
        // 在这里放置你的命令逻辑
        $this->info('命令逻辑尚未实现');

        return self::SUCCESS;
    }
}
```

让我们向命令添加一些逻辑：

```php app/src/Endpoint/Console/CheckStatusCommand.php
#[AsCommand(name: 'check:status')]
final class CheckStatusCommand extends Command
{
    #[Argument(description: '要检查的域名')]
    #[Question(question: '您要检查哪个域名？')]
    private string $domain;

    #[Option(name: 'interval', shortcut: 'i', description: '间隔（分钟）')]
    private int $intervalInMinutes = 5;

    public function __invoke(WorkflowClientInterface $workflowClient): int
    {
        $workflow = $workflowClient->newWorkflowStub(
            WebsiteStatusWorkflow::class,
        );

        $workflowClient->start(
            $workflow,
            $this->domain,
            $this->intervalInMinutes
        );

        return self::SUCCESS;
    }
}
```

现在我们可以使用以下命令运行工作流：

```terminal
php app.php check:status https://spiral.dev -i 5
```

就这样。现在您可以打开 Temporal UI http://127.0.0.1:8233 并查看工作流的执行。

## 计划工作流

[Temporal 计划](https://docs.temporal.io/workflows#schedule) 是传统 cron 作业的任务调度的替代方案，因为计划提供了一种更持久的方式来执行任务，允许深入了解其进度，实现计划和工作流运行的可观察性，并让您启动、停止和暂停它们。

要计划工作流，您需要使用 `Temporal\Client\ScheduleClientInterface` 接口：

```php app/src/Endpoint/Console/CheckStatusCommand.php
<?php

declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;
use Temporal\Client\ScheduleClientInterface;
use Temporal\Client\Schedule;

#[AsCommand(name: 'check:status')]
final class CheckStatusCommand extends Command
{
    #[Argument(description: '要检查的域名')]
    #[Question(question: '您要检查哪个域名？')]
    private string $domain;

    #[Option(name: 'interval', shortcut: 'i', description: '间隔（分钟）')]
    private int $intervalInMinutes = 5;

    public function __invoke(ScheduleClientInterface $client): int
    {
        $client->createSchedule(
            Schedule\Schedule::new()->withAction(
                Schedule\Action\StartWorkflowAction::new(WebsiteStatusWorkflow::class)
                    ->withRetryPolicy(\Temporal\Common\RetryOptions::new()->withMaximumAttempts(3))
                    ->withWorkflowExecutionTimeout('40m')
            )->withSpec(
                Schedule\Spec\ScheduleSpec::new()
                    ->withIntervalList(5 * 60) // 每 5 分钟
                    ->withJitter(60) // 具有 1 分钟的抖动
            ),
        );

        return self::SUCCESS;
    }
}
```

> **注意**
> 您可以在 [Temporal PHP 示例存储库](https://github.com/temporalio/samples-php/tree/master/app/src/Schedule) 中找到如何使用计划的示例。

## 控制台命令

有几个控制台命令可用于管理工作流和活动：

### 列出可用的工作流和活动

要列出所有可用的工作流和活动，请运行以下命令：

```terminal
php app.php temporal:info
```

这是一个示例输出：

```bash
工作流
=========

+-----------------+------------------------------------------------------+------------------+
| 名称            | 类                                                   | 任务队列         |
+-----------------+------------------------------------------------------+------------------+
| fooWorkflow     | Spiral\TemporalBridge\Tests\Commands\Workflow        | worker2          |
|                 | src/Commands/InfoCommandTest.php                     |                  |
| AnotherWorkflow | Spiral\TemporalBridge\Tests\Commands\AnotherWorkflow | default, worker2 |
|                 | src/Commands/InfoCommandTest.php                     |                  |
+-----------------+------------------------------------------------------+------------------+
```

您还可以使用 `--with-activities` 或 `-a` 选项列出所有可用的活动：

```terminal
php app.php temporal:info --with-activities
```

```bash
工作流
=========

+-----------------+------------------------------------------------------+------------------+
| 名称            | 类                                                   | 任务队列         |
+-----------------+------------------------------------------------------+------------------+
| fooWorkflow     | Spiral\TemporalBridge\Tests\Commands\Workflow        | worker2          |
|                 | src/Commands/InfoCommandTest.php                     |                  |
| AnotherWorkflow | Spiral\TemporalBridge\Tests\Commands\AnotherWorkflow | default, worker2 |
|                 | src/Commands/InfoCommandTest.php                     |                  |
+-----------------+------------------------------------------------------+------------------+

活动
==========

+------------------------+---------------------------------------------+------------+
| 名称                   | 类                                        | 任务队列   |
+------------------------+---------------------------------------------+------------+
| fooActivity            | ActivityInterfaceWithWorker::foo            | worker1    |
| bar                    | ActivityInterfaceWithWorker::bar            |            |
+------------------------+---------------------------------------------+------------+
| fooActivity__construct | ActivityInterfaceWithoutWorker::__construct | default    |
| fooActivitybaz         | ActivityInterfaceWithoutWorker::baz         |            |
+------------------------+---------------------------------------------+------------+
```
