# Directory Structure

The framework does not enforce any specific namespace or directory structure. You can freely change them.

## Directories

The default directory structure can be controlled via `mapDirectories` method of Kernel class. By default, all
application directories will be calculated based on `root` using the following pattern:

| Directory | Value                  |
|-----------|------------------------|
| root      | **set by user**        |
| vendor    | **root**/vendor        |
| runtime   | **root**/runtime       |
| cache     | **root**/runtime/cache |
| public    | **root**/public        |
| app       | **root**/app           |
| config    | **app**/config         |
| resources | **app**/resources      |

Some components will declare their own directories such as:

| Component         | Directory  | Value              |
|-------------------|------------|--------------------|
| spiral/views      | views      | **app**/views      |
| spiral/translator | locale     | **app**/locale     |
| spiral/migrations | migrations | **app**/migrations |

## Init Directories

You can specify `root` or any other directory value in your `app.php` file as argument to `App`:

```php
$app = \App\App::create([
    'root' => __DIR__
])->run();
```

For example, we can change runtime directory location:

```php
$app = \App\App::create([
    'root' => __DIR__, 
    'runtime' => sys_get_temp_dir()
])->run();
```

To resolve directory alias within your application use `Spiral\Boot\DirectoriesInterface`:

```php
use Spiral\Boot\DirectoriesInterface;

// ...

function test(DirectoriesInterface $dirs): void
{
    \dump($dirs->get('app'));
    // output: <your root path>/app/
}
```

You can also use the function `directory` inside the global IoC scope (config files, controllers, service code).

```php
function test(): void
{
    \dump(directory('app'));
    // output: <your root path>/app/
}
```

## Namespaces

By default, all skeleton applications use `App` root namespace pointing to `app/src` directory. You can change the base
to any desired namespace in `composer.json`:

```json
{
  "autoload": {
    "psr-4": {
      "App\\": "app/src/"
    }
  }
}
```
