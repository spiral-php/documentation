# 快速入门 — 第一个后台任务

在本指南中，我将逐步引导您使用 Spiral 和 [RoadRunner](https://roadrunner.dev/) 应用服务器创建和运行后台任务。这将使您能够异步执行任务，允许您的应用程序继续运行，而后台任务在后台无缝执行。

让我们深入了解您需要遵循的基本步骤：

## 创建一个任务

为了轻松创建您的第一个任务，请使用脚手架命令：

```terminal
php app.php create:jobHandler PingSite
```

> **注意**
> 在 [基础知识 — 脚手架](../basics/scaffolding.md#job-handler) 部分阅读有关脚手架的更多信息。

执行此命令后，以下输出将确认成功创建：

```output
Declaration of ' [32mPingSiteJob [39m' has been successfully written into ' [33mapp/src/Endpoint/Job/PingSiteJob.php [39m'.
```

现在，让我们将一些逻辑注入到我们新创建的任务处理器中。

以下是一个将 `GET` 请求发送到给定站点的任务示例：

```php app/src/Endpoint/Job/PingSiteJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class PingSiteJob extends JobHandler
{
    public function invoke(HttpClientInterface $client, string $site): void
    {
        $response = $client->request('GET', $site);

        // do something with response
    }
}
```

## 配置

确保在 RoadRunner 服务器中启用任务：

```terminal
./rr serve
```

接下来，我们需要配置我们的应用程序，以便将任务发送到 RoadRunner。 打开 `app/config/queue.php` 文件，并将以下配置添加到该文件中：

```php app/src/Endpoint/Job/PingSiteJob.php
namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class PingSiteJob extends JobHandler
{
    public function invoke(HttpClientInterface $client, string $site): void
    {
        $response = $client->request('GET', $site);

        // do something with response
    }
}
```

这些配置将创建一个新的 `in-memory` 队列来处理您的任务。

## 运行任务

现在，是时候执行我们创建的命令，然后将任务推送到队列了。 使用以下命令：

```terminal
php app.php ping:site "https://google.com"
```

您应该看到以下输出：

```output
Job  [32m3332e595-9774-434c-908c-3c419f80c967 [39m pushed
```

一旦任务被推送到队列中，它将由 RoadRunner 拾取，然后由消费者处理。

完成了！ 恭喜您使用 Spiral 和 RoadRunner 创建了您的第一个后台任务。 有了这个设置，您可以轻松地包含更多任务并异步执行任务，以确保应用程序的平稳运行。

<hr>

## 接下来是什么？

现在，通过阅读一些文章来深入了解基础知识：

*   [队列和任务](../queue/configuration.md)
*   [队列拦截器](../queue/interceptors.md)
*   [创建控制台命令](../console/commands.md)
*   [脚手架](../basics/scaffolding.md)
