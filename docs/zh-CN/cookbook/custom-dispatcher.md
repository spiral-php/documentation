# Cookbook — 自定义调度器

可以使用自定义数据源来调用应用程序内核，例如，**Kafka**、**状态机**事件，或附加到用户定义的中断。在本节中，我们将尝试演示如何编写一个 RoadRunner 服务插件和一个内核调度器，以从该服务消费数据。对于任何有兴趣为 RoadRunner 构建自定义插件或使用 Spiral 构建可扩展和可扩展的 Web 应用程序的人来说，这是一个很好的起点。

在本示例中，我们将每秒向内核发送“tick”。

> **注意**
> 请务必先阅读 [应用服务器](../start/server.md)。 本文假设您精通编写 Golang 代码。

## RoadRunner 服务插件

利用 RoadRunner 性能的一种方法是使用其插件系统，该系统允许您扩展服务器的功能并对其进行自定义以满足您的需求。

在本教程中，我们将向您展示如何创建一个名为“ticker”的简单 RoadRunner 插件，该插件将定期以定义的时间间隔向 PHP worker 发送 ticks。这对于发送定期更新给客户端或运行计划任务等任务非常有用。

### 先决条件

在开始之前，您需要在您的机器上安装以下内容：

- [Go](https://golang.org/doc/install)
- [Velox](https://github.com/roadrunner-server/velox/releases) - 官方 RoadRunner 构建工具。它允许您从 github 和 gitlab 存储库构建自定义 RoadRunner 二进制文件。

> **了解更多**
> 阅读更多关于如何创建 RoadRunner 插件 [here](https://roadrunner.dev/docs/plugins-intro/) 以及如何构建带有自定义插件的二进制文件 [here](https://roadrunner.dev/docs/app-server-build)。

### 插件配置

以下是如何在 `.rr.yaml` 中配置 ticker 插件的示例：

```yaml .rr.yaml
server:
  command: php app.php

ticker:
  interval: 1s
  pool:
    num_workers: 2
```

如您所见，我们的配置允许我们以 `1s, 1m, 10s, ...` 格式定义 ticks 之间的时间间隔，并配置 worker 池。 `interval` 字段指定 ticks 之间要等待的时间。

让我们为我们的服务创建一个配置文件 `config.go`：

```go config.go
package ticker

import (
	"time"
	"github.com/roadrunner-server/sdk/v3/pool"
)

type Config struct {
	Interval time.Duration `mapstructure:"interval"`
	Pool     *pool.Config  `mapstructure:"pool"`
}

func (c *Config) InitDefaults() {
	if c.Pool == nil {
		c.Pool = &pool.Config{}
	}

    // Init default pool settings
	c.Pool.InitDefaults()

	// use default interval 1s when inteval is not defined or defined with wrong value 
	if c.Interval == 0 {
		c.Interval = time.Second
	}
}
```

在 `config.go` 文件中，我们定义了一个名为 `Config` 的结构体来存储插件配置。它有一个 `Interval` 字段用于存储 tick 间隔，以及一个 `Pool` 字段用于存储 worker 池配置。如果未在 `.rr.yaml` 文件中指定这些字段，则 `InitDefaults` 函数会为这些字段设置默认值。 默认间隔设置为 1 秒，默认 worker 池配置设置为 RoadRunner SDK 提供的默认值。

### 插件服务

我们已经定义了 ticker 插件的配置，接下来让我们开始创建插件服务。

插件服务负责管理 worker 并向它们发送 ticks。要创建服务，请创建一个名为 `plugin.go` 的新文件，并将以下代码添加到其中：

```go plugin.go
package ticker

import (
	"context"
	"fmt"
	"sync"
	"time"

	"github.com/roadrunner-server/errors"
	"github.com/roadrunner-server/sdk/v3/payload"
	"github.com/roadrunner-server/sdk/v3/pool"
	"github.com/roadrunner-server/sdk/v3/pool/static_pool"
	"github.com/roadrunner-server/sdk/v3/worker"
	"go.uber.org/zap"
)

type Configurer interface {
	// UnmarshalKey takes a single key and unmarshal it into a Struct.
	UnmarshalKey(name string, out any) error

	// Has checks if config section exists.
	Has(name string) bool
}

// Server creates workers for the application.
type Server interface {
	NewPool(ctx context.Context, cfg *pool.Config, env map[string]string, _ *zap.Logger) (*static_pool.Pool, error)
}

type Pool interface {
	// Workers returns worker list associated with the pool.
	Workers() (workers []*worker.Process)

	// Exec payload
	Exec(ctx context.Context, p *payload.Payload) (*payload.Payload, error)

	// Reset kill all workers inside the watcher and replaces with new
	Reset(ctx context.Context) error

	// Destroy all underlying stack (but let them to complete the task).
	Destroy(ctx context.Context)
}

const (
	rrMode     string = "RR_MODE"
	pluginName string = "ticker"
)

type Plugin struct {
	mu     sync.RWMutex
	cfg    *Config
	server Server
	stopCh chan struct{}
	pool   Pool
}

func (p *Plugin) Init(cfg Configurer, server Server) error {
	// If config file doesn't contain plugin section, ignore it
    if !cfg.Has(pluginName) {
		return errors.E(errors.Disabled)
	}

	// read plugin config
	err := cfg.UnmarshalKey(pluginName, &p.cfg)
	if err != nil {
		return err
	}

	p.cfg.InitDefaults()

	p.stopCh = make(chan struct{}, 1)
	p.server = server

	return nil
}

func (p *Plugin) Serve() chan error {
	errCh := make(chan error, 1)

	var err error
	p.mu.Lock()
    // Create workers pool
	p.pool, err = p.server.NewPool(context.Background(), p.cfg.Pool, map[string]string{rrMode: pluginName}, nil)
	p.mu.Unlock()

	if err != nil {
		errCh <- err
		return errCh
	}

	go func() {
		var numTicks = 0
		var lastTick time.Time
        // Be careful with ticker! You should always stop it
		ticker := time.NewTicker(p.cfg.Interval)
		defer ticker.Stop()

		for {
			select {
			case <-p.stopCh:
				return
			case <-ticker.C:
				p.mu.RLock()
				_, err2 := p.pool.Exec(context.Background(), &payload.Payload{
					Context: []byte(fmt.Sprintf(`{"lastTick": %v}`, lastTick.Unix())),
					Body:    []byte(fmt.Sprintf(`{"tick": %v}`, numTicks)),
				})
				p.mu.RUnlock()
				if err != nil {
					errCh <- err2
					return
				}

				numTicks++
				lastTick = time.Now()
			}
		}
	}()

	return errCh
}

func (p *Plugin) Reset() error {
	p.mu.RLock()
	defer p.mu.RUnlock()

	if p.pool == nil {
		return nil
	}

	ctx, cancel := context.WithTimeout(context.Background(), time.Second*10)
	defer cancel()
	err := p.pool.Reset(ctx)
	if err != nil {
		return err
	}

	return nil
}

func (p *Plugin) Stop() error {
	p.stopCh <- struct{}{}
	return nil
}

func (p *Plugin) Name() string {
	return pluginName
}

func (p *Plugin) Weight() uint {
	return 10
}
```

当 RoadRunner 启动 PHP worker 时，它可以传递 `RR_MODE` 变量的值来指示应使用哪个插件。 然后，Spiral 可以使用该变量的值为当前环境选择合适的调度器。

### 使用 Velox 构建 RoadRunner 二进制文件

接下来，我们将使用 [Velox](https://roadrunner.dev/docs/app-server-build) 来构建带有我们插件的自定义 RoadRunner 二进制文件。

创建一个名为 `plugins.toml` 的新文件，并添加以下配置：

```toml plugins.toml
[velox]
build_args = ['-trimpath', '-ldflags', '-s -X github.com/roadrunner-server/roadrunner/v2/internal/meta.version=v2.12.1.custom -X github.com/roadrunner-server/roadrunner/v2/internal/meta.buildTime=00:00:00']

[roadrunner]
ref = "v2.12.1"

[github]
[github.token]
token = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# ref -> master, commit or tag
[github.plugins]
logger = { ref = "master", owner = "roadrunner-server", repository = "logger" }
server = { ref = "master", owner = "roadrunner-server", repository = "server" }
ticker = { ref = "main", owner = "roadrunner-php", repository = "rr-examples", folder = "ticker" }


[log]
level = "debug"
mode = "development"
```

然后，运行以下命令来构建 RoadRunner 二进制文件：

```terminal
vx build -c plugins.toml -o .
```

## 应用程序调度器

首先，我们需要安装 `spiral/roadrunner-worker` 包：

```terminal
composer require spiral/roadrunner-worker
```

现在我们可以创建我们的调度器：

```php
namespace App\Dispatcher;

use Psr\Container\ContainerInterface;
use Spiral\Boot\DispatcherInterface;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\FinalizerInterface;
use Spiral\RoadRunner\Worker;

final class TickerDispatcher implements DispatcherInterface
{
    public function __construct(
        private readonly EnvironmentInterface $env,
        private readonly FinalizerInterface $finalizer,
        private readonly ContainerInterface $container
    ) {
    }

    public function canServe(): bool
    {
        return $this->env->get('RR_MODE') === 'ticker';
    }

    public function serve(): void
    {
        /** @var Worker $worker */
        $worker = $this->container->get(Worker::class);

        while ($payload = $worker->waitPayload()) {
            $data = \json_decode($payload->body, true);
            
            // Handle tick ... 
        
            // Respond Answer
            $worker->respond(new \Spiral\RoadRunner\Payload('OK'));

            // reset some stateful services
            $this->finalizer->finalize();
        }
    }
}
```

> **了解更多**
> 在 [框架 — 调度器](../framework/dispatcher.md) 部分阅读更多关于调度器的信息。

创建一个 Bootloader 来在内核中注册我们的调度器：

```php app/src/Application/Bootloader/TickerBootloader.php
namespace App\Application\Bootloader;

use App\Dispatcher\TickerDispatcher;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\KernelInterface;

final class TickerBootloader extends Bootloader
{
    public function boot(KernelInterface $kernel, TickerDispatcher $ticker): void
    {
        $kernel->addDispatcher($ticker);
    }
}
```

现在我们可以运行我们的应用程序：

```terminal
./rr serve
```

> **注意：**
> 您可以在 [这里](https://github.com/roadrunner-php/rr-examples) 找到 ticker 插件的代码
