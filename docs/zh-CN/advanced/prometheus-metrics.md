# 高级 — 应用程序指标

作为一名专业人士，您知道跟踪应用程序关键指标的重要性。使用 [Prometheus](https://prometheus.io/)，您可以收集和存储时间序列数据，例如应用程序指标，并使用其强大的查询语言来分析和可视化数据，同时结合 [Grafana](https://grafana.com/) 等工具，节省您从头开始构建仪表板的时间和精力。

![Grafana 仪表板](https://user-images.githubusercontent.com/773481/205066017-ecddefc4-1d07-4428-b3ad-af49baadad0a.png)

Spiral 和 [RoadRunner 指标](https://roadrunner.dev/docs/plugins-metrics) 插件提供了收集应用程序指标并将它们暴露给 Prometheus 的能力。

> **注意**
> 在这里，您可以找到有关 [Prometheus 指标](https://prometheus.io/docs/concepts/data_model/) 的更多信息。

## 安装

首先，您需要安装 [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) 软件包。

> **注意**
> `spiral/roadrunner-bridge` 软件包允许您将 RoadRunner [指标插件](https://roadrunner.dev/docs/lab-metrics) 与 Spiral 一起使用。此软件包提供了指标的 RPC API 和应用程序的启动器。

安装完软件包后，您可以将 `Spiral\RoadRunnerBridge\Bootloader\MetricsBootloader` 添加到启动器列表中：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\MetricsBootloader::class,
        // ...
    ];
}
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\RoadRunnerBridge\Bootloader\MetricsBootloader::class,
    // ...
];
```

在 [框架 — 启动器](../framework/bootloaders.md) 部分阅读更多关于启动器的信息。
:::

::::

## 配置

指标服务不需要在应用程序中进行配置。但是，您必须在 `.rr.yaml` 中激活该服务：

```yaml
rpc:
  listen: tcp://127.0.0.1:6001

# ...

metrics:
  # prometheus client address (path /metrics added automatically)
  address: 127.0.0.1:2112
```

> **注意**
> 您可以在 http://127.0.0.1:2112 查看默认指标

## 使用

### 应用程序指标声明

在 Spiral 应用程序中声明特定于应用程序的指标有两种方法：

:::: tabs
::: tab RoadRunner
使用 `.rr.yaml` 文件：

```yaml .rr.yaml
metrics:
  address: 127.0.0.1:2112

  collect:
    registered_users:
      type: counter
      help: "Total registered users counter."
```

:::

::: tab PHP
在 PHP 代码中声明指标

```php app/src/Application/Bootloader/MetricsBootloader.php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;

class MetricsBootloader extends Bootloader
{
    //...

    public function boot(MetricsInterface $metrics): void
    {
        $metrics->declare(
            'registered_users',
            Collector::counter()->withHelp('Total registered users counter.')
        );
    }
}
```

:::

::::

要从应用程序填充指标，请使用 `Spiral\RoadRunner\Metrics\MetricsInterface`:

```php
use Spiral\RoadRunner\Metrics\MetricsInterface; 

class UserRegistrationHandler
{
    public function __construct(
        private readonly MetricsInterface $metrics
    ) {
    }

    public function handle(User $user): void
    {
        // Store user in database

        $this->metrics->add('registered_users', 1);
    }
}
```

> **查看更多**
> 支持的类型：gauge, counter, summary, histogram。在 [官方 Prometheus 文档](https://prometheus.io/docs/concepts/metric_types/) 中阅读更多关于指标类型的信息。

### 标记指标

使用标记（也称为带标签的）指标允许您向指标附加额外的元数据，这对于过滤、分组和聚合数据非常有用。

**使用带标签的指标的一些好处包括：**

- **更高的粒度**: 您可以向指标附加多个标签，允许您以各种方式对数据进行切片和切块。
- **更好的组织**: 标签可以帮助您对指标进行分组和组织，从而更容易找到和理解您正在寻找的数据。
- **简化的查询**: 您可以使用标签来过滤和聚合您的指标数据，从而更容易从数据中提取有意义的见解。

:::: tabs
::: tab RoadRunner
使用 `.rr.yaml` 文件：

```yaml .rr.yaml
metrics:
  address: 127.0.0.1:2112

  collect:
    registered_users:
      type: histogram
      help: "Total registered users counter."
      labels: [ "type" ]
```

:::

::: tab PHP
在 PHP 代码中声明指标

```php app/src/Application/Bootloader/MetricsBootloader.php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;

class MetricsBootloader extends Bootloader
{
    //...

    public function boot(MetricsInterface $metrics): void
    {
        $metrics->declare(
            'registered_users',
            Collector::counter()->withHelp('Total registered users counter.')->withLabels('type')
        );
    }
}
```

:::

::::

在此示例中，`registered_users` 指标声明了一个名为 `type` 的标签。 当向指标添加数据时，您可以指定 `type` 标签的值，例如 `customer`、`admin` 等。 这允许您在分析指标数据时区分不同类型的用户。

```php
use Spiral\RoadRunner\Metrics\MetricsInterface; 

class UserRegistrationHandler
{
    public function __construct(
        private readonly MetricsInterface $metrics
    ) {
    }

    public function handle(User $user): void
    {
        // Store user in database

        $this->metrics->add('registered_users', 1, ['customer']);
        
        // or
        
        $this->metrics->add('registered_users', 1, ['admin']);
    }
}
```

## 装饰器

### 重试发送指标

有时，您可能会遇到将指标发送到 RoadRunner 指标插件的问题。 连接可能会断开，或者插件可能会暂时关闭。 对于此类情况，您可以使用重试装饰器。 它允许您指定您想重试的次数和频率。

以下是使用重试装饰器的示例：

```php
$factory = new \Spiral\RoadRunner\Metrics\MetricsFactory();
$rpc = $container->get(\Spiral\Goridge\RPC\RPCInterface::class);

$metrics = $factory->create($rpc, new \Spiral\RoadRunner\Metrics\MetricsOptions(
    retryAttempts: 3,
    retrySleepMicroseconds: 50,
));
```

### 抑制异常

有时，您可能不希望在发送指标时发生错误阻止您的应用程序。 例如，如果您正在处理付款，则指标错误不应中断交易。 您可以抑制这些错误。

以下是抑制异常的示例：

```php
$factory = new \Spiral\RoadRunner\Metrics\MetricsFactory();
$rpc = $container->get(\Spiral\Goridge\RPC\RPCInterface::class);

$metrics = $factory->create(
    $rpc,
    new \Spiral\RoadRunner\Metrics\MetricsOptions(
        suppressExceptions: true,
    ),
));
```

或者，使用 `Spiral\RoadRunner\Metrics\SuppressExceptionsMetrics` 装饰器：

```php
$metrics = new \Spiral\RoadRunner\Metrics\SuppressExceptionsMetrics(
    $container->get(\Spiral\RoadRunner\Metrics\MetricsInterface::class),
));
```

### 在容器中注册装饰器

与其直接在应用程序代码中定义装饰器，不如在容器中设置它们更清晰。

这是一个简单的指南：

```php app/src/Application/Bootloader/MetricsBootloader.php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;
use Spiral\Goridge\RPC\RPCInterface;

class MetricsBootloader extends Bootloader
{
    const SINGLETONS = [
        MetricsInterface::class => [self::class, 'createMetrics'],
    ];
    
    private function createMetrics(RPCInterface $rpc): MetricsInterface
    {
        $factory = new \Spiral\RoadRunner\Metrics\MetricsFactory();
        
        return $factory->create(
            $rpc,
            new \Spiral\RoadRunner\Metrics\MetricsOptions(
                suppressExceptions: true,
                retryAttempts: 3,
                retrySleepMicroseconds: 50,
            ),
        ));
    }
}
```

> **注意**
> 不要忘记在应用程序内核中注册 `MetricsBootloader`。

---

## 示例应用程序

有一个很好的例子 [**演示票务预订系统**](https://github.com/spiral/ticket-booking) 应用程序是基于 Spiral 框架构建的，它遵循微服务的原则，并允许开发人员创建可重用的、独立的且易于维护的组件。

在这个演示应用程序中，您可以找到使用 RoadRunner 指标插件的示例。

总的来说，我们的演示票务预订系统是一个很好的例子，说明了如何使用 Spiral 和其他工具来构建现代且高效的应用程序。 我们希望您在使用它并了解有关我们的框架和我们使用的其他工具的功能时玩得开心。

**祝您（假）购票愉快！**
