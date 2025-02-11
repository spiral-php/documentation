# 高级功能 — 应用遥测

Spiral 是一个用于构建微服务的强大工具。其关键特性之一是 `spiral/telemetry` 组件，它使您能够收集并将应用程序指标发送到遥测服务器或日志。该组件为收集性能数据和监控您的微服务提供了一个灵活且可靠的解决方案。

![OpenTelemetry](https://user-images.githubusercontent.com/773481/213914208-cd944ca8-f218-4baf-8a54-5a4e42a1ed40.jpg)

收集到的跟踪信息随后可以发送到第三方服务进行渲染，从而清晰且详细地可视化您的微服务的性能。

## 使用示例

当客户在网站上下订单时，请求将从前端服务、通过订单处理服务、到库存管理服务，最后到发货服务进行跟踪。

您可以使用**跟踪 ID**来关联与同一请求相关的所有跟踪信息，这样您就可以看到请求的整个路径，以及每个服务如何处理它。通过这些数据，您可以监控执行时间，如果任何服务出现延迟，他们可以通过查看每个服务的跟踪信息进行进一步调查。您还可以监控每个服务正在处理的请求数量，并查看是否有任何服务过载或未充分利用。

此外，您还可以跟踪数据库查询以识别影响系统整体性能的慢速查询。并且，您可以跟踪外部服务调用以识别平台依赖的第三方 API 的任何问题。

## OpenTelemetry 集成

默认情况下，该组件使用 `null` 驱动程序，不执行任何操作。但是，它还通过 [spiral/otel-bridge](https://github.com/spiral/otel-bridge) 包提供了与 [OpenTelemetry](https://opentelemetry.io/) 服务的集成。这允许您在请求跨越微服务时对其进行跟踪，使用通过标头从一个服务传递到下一个服务的**跟踪 ID**。这使您能够全面了解请求是如何处理的，以及不同的微服务是如何交互的。

### 安装

要安装 `spiral/otel-bridge` 包，您可以使用以下命令：

```terminal
composer require spiral/otel-bridge open-telemetry/exporter-otlp
```

> **注意**
> 在我们的示例中，我们使用 `open-telemetry/exporter-otlp` 包将跟踪信息发送到 OpenTelemetry 收集器。
> 如果您想使用不同的导出器，可以在 OpenTelemetry 文档中查看 [导出器部分](https://opentelemetry.io/docs/instrumentation/php/exporters/)。

安装完该包后，您需要在应用程序的内核中注册 bootloader：

:::: tabs

::: tab 使用方法

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\OpenTelemetry\Bootloader\OpenTelemetryBootloader::class,
        // ...
    ];
}
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::: tab 使用常量

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\OpenTelemetry\Bootloader\OpenTelemetryBootloader::class,
    // ...
];
```

在 [框架 — Bootloaders](../framework/bootloaders.md) 部分阅读更多关于 bootloaders 的信息。
:::

::::

### 配置

要完全配置该包，您需要使用适当的设置更新应用程序的 `.env` 文件。

```dotenv .env
# Telemetry driver [log, null, otel]
TELEMETRY_DRIVER=otel

# OpenTelemetry
OTEL_SERVICE_NAME=php # 您的应用程序名称
OTEL_TRACES_EXPORTER=otlp
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://127.0.0.1:4318
OTEL_PHP_TRACES_PROCESSOR=simple
```

> **查看更多**
> 您可以在 [OpenTelemetry 文档](https://opentelemetry.io/docs/concepts/sdk-configuration/general-sdk-configuration/) 中找到有关配置选项的更多信息。

要运行 [OpenTelemetry 收集器](https://opentelemetry.io/docs/collector/) 服务器和 [Zipkin](https://zipkin.io/) 跟踪系统，您可以使用提供的示例 `docker-compose.yaml` 文件：

```yaml docker-compose.yaml
version: "3.6"

services:
  collector:
    image: otel/opentelemetry-collector-contrib
    command: [ "--config=/etc/otel-collector-config.yml" ]
    volumes:
      - ./otel-collector-config.yml:/etc/otel-collector-config.yml
    ports:
      - "4318:4318"

  zipkin:
    image: openzipkin/zipkin-slim
    ports:
      - "9411:9411"
```

以及 `otel-collector-config.yml` 配置文件

```yaml otel-collector-config.yml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:
    timeout: 1s

exporters:
  logging:
    loglevel: debug

  zipkin:
    endpoint: "http://zipkin:9411/api/v2/spans"

  datadog:
    api:
      site: datadoghq.eu
      key: # 您的 datadog api key

  otlp:
    endpoint: https://otlp.eu01.nr-data.net:443
    headers:
      api-key: # 您的 new relic api key

service:
  pipelines:
    traces:
      receivers: [ otlp ]
      processors: [ batch ]
      # 在这里，您可以设置要发送跟踪信息的导出器
      exporters: [ zipkin, datadog, otlp, logging ]
```

您还应该配置 RoadRunner 将跟踪信息发送到 OpenTelemetry 收集器服务器：

```yaml .rr.yaml
http:
  address: 0.0.0.0:8080
  middleware: [ "otel" ]
  otel:
    insecure: true
    compress: false
    client: http
    exporter: otlp
    service_name: rr-blog # 您的应用名称
    service_version: 1.0.0 # 您的应用版本
    endpoint: 127.0.0.1:4318 # otel 收集器服务器地址
```

这将启用集成，并允许您开始使用 OpenTelemetry 服务跟踪您的应用程序中的请求。

#### Monolog 集成

该组件不需要在应用程序内进行任何特定的配置，但它提供了配置 Monolog 以将跟踪上下文添加到日志消息中的能力。这可以通过将 `\Spiral\Telemetry\Monolog\TelemetryProcessor::class` 添加为 `monolog.php` 配置文件中的一个处理器来实现。

```php app/config/monolog.php
return [
    ...

    'processors' => [
        'default' => [
            \Spiral\Telemetry\Monolog\TelemetryProcessor::class,
        ],
    ],
];
```

这允许将**跟踪 ID**与日志信息一起存储。这使得更容易找到特定日志的跟踪信息并调查问题，因为它允许关联日志和跟踪数据。

## 用法

该组件提供了一个 `Spiral\Telemetry\TracerInterface` 接口，该接口可用于将跟踪信息发送到收集器。

使用示例：

```php
use Spiral\Telemetry\TracerInterface;
use Spiral\Telemetry\TraceKind;
use Spiral\Telemetry\SpanInterface;

$tracer = $this->container->get(TracerInterface::class);
$url = 'https://example.com';

$result = $tracer->trace(
    name: 'some.function'
    callback: static function(
        SpanInterface $span,
        HttpClientInterface $httpClient
    ) use($url): string {
        // 回调函数内部的代码将在 span 上下文中执行，并且关于 span 的信息将被发送到收集器
        
        $response = $httpClient->get($url);
        
        // 将添加到 span 对象中的属性
        $span->setAttribute('http.response.code', $response->getStatusCode());
        $span->setAttribute('http.response.length', \strlen($response->getContent()));
        
        return $response->getContent();
    },
    attributes: [
        'http.url' => $url,
    ],
    scoped: true,
    traceKind: TraceKind::CLIENT,
);
```

`trace` 方法使用以下参数调用：

- `name` - span 的名称。此名称将用于在跟踪中标识 span。
- `callback` - 将在 span 上下文中执行的回调函数。回调函数将接收当前的 span 对象和容器作为参数。回调函数可以返回任何值。您可以在回调函数中使用依赖注入，这对于注入服务或您的函数需要执行的其他依赖项很有用。
- `attributes` - 将添加到 span 对象中的属性。
- `scoped` - 如果为 `true`，则回调函数内部的所有 span 都将与当前的 span 相关。
- `traceKind` - 一个指示 span 类型的常量（client, server, etc）。

传递给回调函数的 `SpanInterface` 作为一个参数，可以用于操作当前的 span：

- `updateName(string $name)`: 更新当前 span 的名称
- `setStatus(string|int $code, string $description = null)`: 设置当前 span 的状态
- `setAttributes(array $attributes)`: 设置当前 span 的属性
- `setAttribute(string $name, mixed $value)`: 在当前 span 上设置一个属性

这些方法可用于向 span 添加更多信息，例如属性、状态，以及更新 span 的名称。这使您可以向跟踪添加更多上下文，并获得有关回调函数内部代码执行的更多信息。

### 发送跟踪上下文

跟踪上下文是一组键值对，其中包含有关当前跟踪的信息，例如跟踪 ID、span ID 和其他属性。此上下文用于将构成跟踪的多个 span 链接在一起。

当您想将跟踪上下文发送到另一个应用程序时，您可以从 `Spiral\Telemetry\TracerInterface` 获取它，方法是调用 `getContext()` 方法。此方法返回跟踪上下文的关联数组。然后，您可以循环遍历上下文，并将键值对添加为发送到另一个应用程序的响应的标头。

```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    $response = $responseFactory->createResponse();

    $tracer = $this->container->get(TracerInterface::class);
    
    foreach ($tracer->getContext() as $key => $value) {
        $response = $response->withHeader($key, $value);
    }
    
    return $response;
}
```

这允许另一个应用程序访问跟踪上下文，并将其链接到请求所属的跟踪。这使得可以跨不同的服务跟踪请求，这有助于理解请求的流程并识别问题。

### 从上下文创建跟踪

当您从另一个应用程序获取跟踪上下文并希望基于它创建跟踪时，可以使用 `Spiral\Telemetry\TracerFactoryInterface`。此接口提供了一个 createTracer 方法，该方法接受一个上下文键值对数组，并返回 `Spiral\Telemetry\TracerInterface` 的一个实例。此实例可用于创建新的 span 并将它们链接到该上下文所属的跟踪。

```php
public function handle(ServerRequestInterface $request): ResponseInterface
{
    $tracerFactory = $this->container->get(\Spiral\Telemetry\TracerFactoryInterface::class);
    $tracer = $tracerFactory->make($request->getHeaders());
    
   $response = $tracer->trace(
        name: \sprintf('%s %s', $request->getMethod(), (string)$request->getUri()),
        callback: $callback,
        attributes: [
            'http.method' => $request->getMethod(),
            'http.url' => $request->getUri(),
            'http.headers' => $request->getHeaders(),
        ],
        scoped: true,
        traceKind: TraceKind::SERVER
    );
    
    ...
}
```

## 示例应用程序

有一个很好的例子 [**演示票务预订系统**](https://github.com/spiral/ticket-booking) 应用程序是基于 Spiral 框架构建的，它遵循微服务的原则，并允许开发人员创建可重用、独立且易于维护的组件。

在这个演示应用程序中，您可以找到使用 OpenTelemetry 的示例。

总的来说，这是一个很好的例子，说明了如何使用 Spiral 和其他工具来构建现代且高效的应用程序。我们希望您在使用它并了解 Spiral 和我们使用的其他工具的功能时，能获得愉快的体验。

**祝您购物愉快（虚构的）！**
