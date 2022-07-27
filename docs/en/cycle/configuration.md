# Cycle ORM - Installation and Configuration

Cycle ORM is included into default [Web skeleton](https://github.com/spiral/app).

To enable full ORM integration in other skeletons, use following bootloaders:

```php
[
    Spiral\Bootloader\Cycle\CycleBootloader::class,
    Spiral\Bootloader\Cycle\ProxiesBootloader::class,
    Spiral\Bootloader\Cycle\AnnotatedBootloader::class,
]
```

The following components required in `composer.json`:

```json
{
  "cycle/orm": "^1.0",
  "cycle/proxy-factory": "^1.0",
  "cycle/annotated": "^2.0",
  "cycle/migrations": "^1.0"
}
```

> You can enable and disable ORM extensions separately.

## Configuration

ORM does not require any configuration. You can specify connections and drivers in the DBAL Component configuration.
