# Extensions - Sentry
You can automatically send the exceptions to the Sentry.

## Installation
To install the extension:

```bash
composer require spiral/sentry-bridge
```

Activate the bootloader `Spiral\Sentry\Bootloader\SentryBootloader`:

```php
protected const LOAD = [
    // ...
    Spiral\Sentry\Bootloader\SentryBootloader::class,
    // ...
];
```

> Remove default `Spiral\Bootloader\SnapshotsBootloader`.

The component will look at `SENTRY_DSN` env value.

## Additional Data
To expose current application logs, PSR-7 request state, etc. enable additional
debug extensions:

```php
protected const LOAD = [
    // ...
    Spiral\Sentry\Bootloader\SentryBootloader::class,
  
    // ...
    
    // at the end of the chain
    Spiral\Bootloader\DebugBootloader::class,
    Spiral\Bootloader\Debug\LogCollectorBootloader::class,
    Spiral\Bootloader\Debug\HttpCollectorBootloader::class,   
];
```
