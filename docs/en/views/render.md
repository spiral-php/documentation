# Views - Rendering Views

To render view, you must obtain an instance of `Spiral\Views\ViewsInterface`.

> **Note**
> The component is available via prototyped property `views`.

## Rendering

To render view in the controller or other service simply invoke the `render` method of `ViewsInterface`. The view name
does not need to include extension or namespace (default to be used).

```php
use Spiral\Views\ViewsInterface;

// ...

public function index(ViewsInterface $views): string
{
    return $views->render('home');
}
```

To render the view with passed data, use the second array argument:

```php
use Spiral\Views\ViewsInterface;

// ...

public function index(ViewsInterface $views): string
{
    return $views->render('home', [
        'key' => 'value'
    ]);
}
```

#### Namespaces

To render view from specific namespace prepend it to view name using `:` separator:

```php
return $views->render('namespace:home', [
    'key' => 'value'
]);
```

## View Object

In some cases it might be more performant to cache the view in stateless component, obtain view
object (`Spiral\Views\ViewInterface`) using method `get` of `ViewsInterface`:

```php
$view = $views->get('home');
```  

To render obtained view call `render` method:

```php
return $view->render([
    'key' => 'value'
]);
```

## Global variables

You can add global variables that will be available in all views. There are several ways to add a global variable.
Using configuration file `app/config/views.php`.

```php
return [
    'globalVariables' => [
        'some_var' => env('SOME_VALUE'),
        'other_var' => 'other_value'
    ]
];
```

Or you can use `Spiral\Views\GlobalVariablesInterface`.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Views\GlobalVariablesInterface ;
use Spiral\Boot\EnvironmentInterface;

class AppBootloader extends Bootloader 
{
    public function boot(GlobalVariablesInterface $vars, EnvironmentInterface $env): void
    {
         $vars->set('some_var', $env->get('SOME_VALUE'));
         $vars->set('other_var', 'other_value');
    }
}
```

After that, you can use the variable in the view.

```php
<!doctype html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>Document</title>
</head>
<body>
    <?=$some_var?>

    <?=$other_var?>
</body>
</html>
```
