# Framework - Kernel and Environment

Each spiral build will contain a kernel object with a set of application-specific services. Unlike Symfony,
The Spiral needs only one Kernel for all the dispatching methods (HTTP, Queue, GRPC, Console, etc). The kernel will
select the dispatching method automatically, based on connected `Spiral\Boot\DispatcherInterface` instances.

> **Note**
> The base kernel implementation located in `spiral/boot` repository.

## Kernel Responsibilities

The `Spiral\Boot\AbstractKernel` class only responsible for the following aspects of your application:

- container initialization via a set of application-specific bootloaders
- environment and directory structure initialization
- selection of appropriate dispatching method

To create your kernel extends `Spiral\Boot\AbstractKernel`.

```php
namespace App;

use Spiral\Boot\AbstractKernel;
use Spiral\Boot\Exception\BootException;

class MyApp extends AbstractKernel
{
    protected const LOAD = [
        // bootloaders to initialize
    ];

    public function get(string $service)
    {
        return $this->container->get($service);
    }

    protected function bootstrap()
    {
        // custom initialization code
        // invoked after all bootloaders are loaded
    }

    protected function mapDirectories(array $directories): array
    {
        if (!isset($directories['root'])) {
            throw new BootException('Missing required directory `root`');
        }

        if (!isset($directories['app'])) {
            $directories['app'] = $directories['root'] . '/app/';
        }

        return \array_merge(
            [
                // public root
                'public'    => $directories['root'] . '/public/',

                // vendor libraries
                'vendor'    => $directories['root'] . '/vendor/',

                // data directories
                'runtime'   => $directories['root'] . '/runtime/',
                'cache'     => $directories['root'] . '/runtime/cache/',

                // application directories
                'config'    => $directories['app'] . '/config/',
                'resources' => $directories['app'] . '/resources/',
            ],
            $directories
        );
    }
}
```

> **Note**
> The `Spiral\Framework\Kernel` defines the default directory map.

## Kernel initialization

To init the kernel invoke static method `create`:

```php
$myapp = MyApp::create(
    [
        'root' => __DIR__
    ],
    false // do not mount error handler
);

$myapp->run(null); // use default env 

\dump($myapp->get(\Spiral\Boot\DirectoriesInterface::class)->getAll());
```

## Environment

Use `Spiral\Boot\EnvironmentInterface` to access the list of ENV variables. By default, the framework relies on
system-level environment values. To redefine env values while initializing the kernel pass custom `EnvironmentInterface`
to the `create` method.

```php
use \Spiral\Boot\Environment;

$myapp = MyApp::create(
    [
        'root' => __DIR__
    ],
    false // do not mount error handler
);

$myapp->run(new Environment(['key' => 'value']));

\dump($myapp->get(\Spiral\Boot\EnvironmentInterface::class)->getAll());
```

> **Note**
> Such an approach can be used to bootstrap the application for testing purposes.
