# Application Metrics

You can expose some of the application metrics using the [prometheus](https://prometheus.io/) service embedded to the
[RoadRunner application server](https://roadrunner.dev/docs/plugins-metrics/2.x/en).

To install the component:

```bash
composer require spiral/roadrunner-metrics
```

## Framework Integration

Create a bootloader

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\RoadRunner\Metrics\Metrics;
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Environment;
use Goridge\RPC\RPC;

class MetricsBootloader extends Bootloader
{
    const SINGLETONS = [
        MetricsInterface::class => [self::class, 'initMetrics'],
    ];
    
    private function initMetrics(): MetricsInterface
    {
        return new Metrics(
            RPC::create(Environment::fromGlobals()->getRPCAddress())
        );
    }
}
```

Make sure to add `App\Bootloader\MetricsBootloader` to your App class:

```php
protected const LOAD = [
    // ...
    App\Bootloader\MetricsBootloader::class
];
```

## Configuration

Metrics service does not require configuration in the application. However, you must activate this service
in `.rr.yaml`:

```yaml
metrics:
  # prometheus client address (path /metrics added automatically)
  address: localhost:2112
```

> **Note**
> You can view defaults metrics on http://localhost:2112/metrics

## Custom Application metrics

You can also publish application-specific metrics. First, you have to register a metric in your configuration file:

```yaml
metrics:
  address: localhost:2112
  collect:
    app_metric_counter:
      type: counter
      help: "Application counter."
```

or declare metrics in PHP code

```php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;

class MetricsBootloader extends Bootloader
{
    //...

    public function boot(MetricsInterface $metrics): void
    {
        $metrics->declare(
            'app_metric_counter',
            Collector::counter()->withHelp('Application counter.')
        );
    }
}
```

> **Note**
> Supported types: gauge, counter, summary, histogram.

To populate metric from application use `Spiral\RoadRunner\Metrics\MetricsInterface`:

```php
use Spiral\RoadRunner\Metrics\MetricsInterface; 

// ...

public function index(MetricsInterface $metrics)
{
    $metrics->add('app_metric_counter', 1);
}
```

> You can call MetricsInterface in middleware.

## Tagged metrics

You can use tagged (labels) metrics to group values:

```yaml
metrics:
  address: localhost:2112
  collect:
    app_type_duration:
      type: histogram
      help: "Application counter."
      labels: [ "type" ]
```


or declare metrics in PHP code

```php
use Spiral\RoadRunner\Metrics\MetricsInterface;
use Spiral\RoadRunner\Metrics\Collector;

class MetricsBootloader extends Bootloader
{
    //...

    public function boot(MetricsInterface $metrics): void
    {
        $metrics->declare(
            'app_metric_counter',
            Collector::counter()->withHelp('Application counter.')->withLabels('type')
        );
    }
}
```

You should specify values for your labels while pushing the metric:

```php
use Spiral\RoadRunner\MetricsInterface; 

// ...

public function index(MetricsInterface $metrics)
{
    $metrics->add('app_type_duration', 0.5, ['some-type']);
}
```
