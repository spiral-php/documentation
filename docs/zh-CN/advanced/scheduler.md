# 高级特性 — 任务调度

[`spiral-packages/scheduler`](https://github.com/spiral-packages/scheduler) 包提供了一个简单的 API，用于通过计划运行 cron 作业任务。它可以轻松地与基于 Spiral 的项目集成。

## 安装

要安装该包，您可以使用以下命令：

```terminal
composer require spiral-packages/scheduler
```

安装完该包后，您需要将该包的引导程序添加到您的应用程序中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Scheduler\Bootloader\SchedulerBootloader::class,
        // ...
    ];
}
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Scheduler\Bootloader\SchedulerBootloader::class,
    // ...
];
```

在 [Framework — Bootloaders](../framework/bootloaders.md) 部分阅读有关引导程序的更多信息。
:::

::::

要运行计划任务，您有两种选择：

1. 将一个 cron 配置条目添加到您的服务器，该条目每分钟运行 `schedule:run` 命令。

```bash
* * * * * cd /path-to-your-project && php app.php schedule:run >> /dev/null 2>&1
```

2. 通过使用 `schedule:work` 命令通过 RoadRunner 运行计划任务。此命令将在前台运行，并每分钟调用调度程序，直到您终止该命令：

```terminal
php app.php schedule:work
```

您可以使用任何守护进程来保持该进程在后台模式下运行，例如 supervisord。

## 用法

要在您的项目中使用 Scheduler 包，您可以创建一个新的 `SchedulerBootloader` 引导程序。

> **警告**
> 不要忘记在您的应用程序中注册 `SchedulerBootloader`！

此引导程序将负责管理 cron 作业任务：

```php app/src/Application/Bootloader/SchedulerBootloader.php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Scheduler\Schedule;
use Psr\Log\LoggerInterface;
use Spiral\Boot\DirectoriesInterface;

final class SchedulerBootloader extends Bootloader
{
    public function boot(Schedule $schedule, DirectoriesInterface $dirs): void
    {
        // 按名称运行控制台命令
        $schedule->command('ping', ['https://google.com'])
            ->everyFiveMinutes()
            ->withoutOverlapping()
            ->appendOutputTo($dirs->get('runtime').'logs/cron.log');
            
        // 按类名运行控制台命令
        $schedule->command(PingCommand::class, ['https://google.com'])
            ->everyFiveMinutes()
            ->withoutOverlapping()
            ->appendOutputTo($dirs->get('runtime').'logs/cron.log');
            
        // 运行可调用任务
        $schedule->call('Ping url', static function (LoggerInterface $logger, string $url) {
            $headers = @get_headers($url);
            $status = $headers && \strpos($headers[0], '200');

            $logger->info(\sprintf('URL: %s %s', $url, $status ? 'Exists' : 'Does not exist'));

            return $status;
        }, ['url' => 'https://google.com'])
           ->everyFiveMinutes()
           ->withoutOverlapping();
    }
}
```

此示例演示了如何使用 Scheduler 包来安排不同类型的命令以在特定的时间间隔运行，以及如何配置计划任务以防止重叠并记录输出。

现在，您可以检查是否已注册 cron 作业任务：

```terminal
php app.php schedule:list
```

```output
+--------------------------------------------------------+-------------+-------------+----------------------------+
| [32m Command                                                 [39m| [32m Interval     [39m| [32m Description  [39m| [32m Next Due                    [39m|
+--------------------------------------------------------+-------------+-------------+----------------------------+
| /usr/bin/php8.1 app.php ping 'https://google.com'      | */5 * * * * |             | 2023-01-12 12:55:00 +00:00 |
| /usr/bin/php8.1 app.php ping:site 'https://google.com' | */5 * * * * | Ping site   | 2023-01-12 12:55:00 +00:00 |
| callback: 'url'                                        | */5 * * * * | Ping url    | 2023-01-12 12:55:00 +00:00 |
+--------------------------------------------------------+-------------+-------------+----------------------------+
```

### 回调

您可以使用 `before`、`then`、`when` 和 `skip` 方法注册回调。

#### Before 回调

`before` 方法可用于注册一个回调，该回调将在计划任务执行之前被调用。此回调可用于在任务运行之前执行任何必要的准备或设置。

```php
$schedule->command('backup:run')
   ->everyFiveMinutes()
   ->before(static fn(Notifier $notifier) => $notifier->send('Starting backup...'));
   -> ...;
```

#### Then 回调

`then` 方法可用于注册一个回调，该回调将在计划任务执行之后被调用。此回调可用于在任务运行之后执行任何必要的清理或附加处理。

```php
$schedule->command('backup:run')
   ->everyFiveMinutes()
   ->then(static fn(Notifier $notifier) => $notifier->send('Backup completed'));
   -> ...;
```

#### When 回调

`when` 方法可用于注册一个回调，该回调将被调用以确定是否应该执行计划任务。此回调可用于检查任何可能阻止任务运行的约束条件。如果 when 回调返回 `false`，则不会执行计划任务。

```php
$schedule->command('email:digest')
   ->everyFiveMinutes()
   ->when(static fn(HolidaysCalendar $calendar) => !$calendar->isHoliday());
   -> ...;
```

#### Skip 回调

`skip` 方法可用于注册一个回调，该回调将被调用以确定是否应该跳过计划任务。此回调可用于检查任何可能阻止任务运行的条件。如果 skip 回调返回 `true`，则不会执行计划任务。

```php
$schedule->command('email:digest')
   ->everyFiveMinutes()
   ->skip(static fn(HolidaysCalendar $calendar) => $calendar->isHoliday());
   -> ...;
```

### 后台任务

您可以使用 `runInBackground()` 方法在后台运行任务：

```php
$schedule->command('ping', ['https://google.com'])
   ->everyFiveMinutes()
   ->runInBackground()
   -> ...;
```

> **警告**
> 不要让可调用任务在后台运行。这将导致错误。

### 防止任务重叠

当使用 `withoutOverlapping()` 方法时，该包将在调度其新实例之前检查该命令或函数是否已在运行。如果该命令或函数的一个实例已在运行，则不会调度新实例，并且将被跳过。这有助于防止同一命令或函数的多个实例同时运行并可能导致问题。

```php
$schedule->command('ping', ['https://google.com'])
   ->everyMinute()
   ->withoutOverlapping()
   -> ...;
```

除了该方法的默认行为外，您还可以指定一个自定义的锁定过期时间（以分钟为单位）。这可以通过将一个整数值作为参数传递给该方法来完成。

```php
$schedule->command('ping', ['https://google.com'])
   ->everyMinute()
   ->withoutOverlapping(60)
   -> ...;
```

> **注意**
> 默认情况下，锁定过期时间为 24 小时。

### Cron 表达式

您可以使用 expression 参数为任务分配自定义的 cron 表达式。

例如，以下代码为 `email:send` 命令分配了 cron 表达式 `* * * * *`（它对应于每分钟运行）：

```php
$schedule->command('email:send', expression: new \Cron\CronExpression('* * * * *'));
```

此外，您可以使用辅助方法以更易于阅读的方式定义您的计划任务。这些方法包括：

#### 分钟

| 方法                  | 描述                           |
|-------------------------|---------------------------------------|
| `everyMinute()`         | 每分钟: `* * * * *`             |
| `everyEvenMinute()`     | 每偶数分钟: `*/2 * * * *`      |
| `everyFiveMinutes()`    | 每五分钟: `*/5 * * * *`     |
| `everyTenMinutes()`     | 每十分钟: `*/10 * * * *`     |
| `everyFifteenMinutes()` | 每十五分钟: `*/15 * * * *` |
| `everyThirtyMinutes()`  | 每三十分钟: `0,30 * * * *`  |

> **注意**
> 更多关于如何安排任务以在特定分钟运行的示例，你可以在 [这里](https://github.com/butschster/CronExpressionGenerator#manipulate-hours) 找到。

#### 小时

| 方法                 | 描述                                          |
|------------------------|------------------------------------------------------|
| `hourly()`             | 每小时的 00 分钟: `0 * * * *`                |
| `hourlyAt(15)`         | 每小时的指定分钟: `15 * * * *`     |
| `hourlyAt(15, 30, 45)` | 每小时的 15、30、45 分钟: `15,30,45 * * * *` |
| `everyTwoHours()`      | 每天的 00:00: `0 0 * * *`                      |

> **注意**
> 更多关于如何安排任务以按小时运行的示例，你可以在 [这里](https://github.com/butschster/CronExpressionGenerator#manipulate-hours) 找到。

#### 天

| 方法                       | 描述                                              |
|------------------------------|----------------------------------------------------------|
| `daily()`                    | 每天的 00:00: `0 0 * * *`                          |
| `daily(13)`                  | 每天的指定时间: `0 13 * * *`            |
| `daily(3, 15, 23)`           | 每天的 03:00、15:00、23:00: `0 3,15,23 * * *`      |
| `twiceDaily(1, 13)`          | 每天的 01:00、13:00: `0 1,13 * * *`                |
| `lastDayOfMonth()`           | 每月的最后一天 00:00: `0 0 L * *`        |
| `lastDayOfMonth(12)`         | 每月的最后一天 12:00: `0 12 L * *`       |
| `lastDayOfMonth(12, 30)`     | 每月的最后一天 12:30: `30 12 L * *`      |
| `lastWeekdayOfMonth()`       | 每月的最后一个工作日 00:00: `0 0 LW * *`   |
| `lastWeekdayOfMonth(12)`     | 每月的最后一个工作日 12:00: `0 12 LW * *`  |
| `lastWeekdayOfMonth(12, 30)` | 每月的最后一个工作日 12:30: `30 12 LW * *` |

> **注意**
> 更多关于如何安排任务以按天运行的示例，你可以在 [这里](https://github.com/butschster/CronExpressionGenerator#manipulate-days) 找到。

#### 星期几

| 方法     | 描述                       |
|------------|-----------------------------------|
| `weekly()` | 每周一: `0 0 * * 0` |

> **注意**
> 更多关于如何安排任务以每周运行的示例，你可以在 [这里](https://github.com/butschster/CronExpressionGenerator#manipulate-days-of-week) 找到。

### 月份

| 方法                  | 描述                                               |
|-------------------------|-----------------------------------------------------------|
| `monthly()`             | 每月 1 日 00:00: `0 0 1 * *`             |
| `monthly(12)`           | 每月 1 日 12:00: `0 12 1 * *`            |
| `monthly(12, 30)`       | 每月 1 日 12:30: `30 12 1 * *`           |
| `monthlyOn(15, 12)`     | 每月 15 日 12:00: `0 12 15 * *`          |
| `monthlyOn(15, 12, 30)` | 每月 15 日 12:30: `30 12 15 * *`         |
| `quarterly()`           | 每季度 yyyy-01,03,06,09-01 00:00: `0 0 1 1-12/3 *` |
| `yearly()`              | 每年 yyyy-01-01 00:00: `0 0 1 1 *`                  |

> **注意**
> 更多关于如何安排任务以按月运行的示例，你可以在 [这里](https://github.com/butschster/CronExpressionGenerator#manipulate-months) 找到。

## 控制台命令

#### 列出已调度的作业

使用 `schedule:list` 命令列出所有已调度的作业：

```terminal
php app.php schedule:list
```

#### 启动计划任务工作进程

使用 `schedule:work` 命令启动计划任务工作进程：

```terminal
php app.php schedule:work
```
