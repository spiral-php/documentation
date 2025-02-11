# 视图 — 纯 PHP 模板

你可以使用纯 PHP 文件作为你的视图，无需任何编译或缓存层。此引擎默认启用，并在 Web 应用程序骨架中可用。

## 创建视图

要创建一个纯 PHP 视图，只需将一个扩展名为 `.php` 的文件放入 `views` 目录中。模板可以通过其文件名进行渲染：

```php test.php
echo "hello world";
```

```php
public function index(ViewsInterface $views): string
{
    return $views->render('test'); // 无需指定扩展名
}
```

所有传递的参数都将作为 PHP 变量可用：

```php
public function index(ViewsInterface $views): string
{
    return $views->render('test', ['name' => 'Antony']);
}
```

其中 `test.php`：

```php test.php
Hello , <?= $name ?>!
```

## 容器

你可以在你的视图文件中通过 `$this->container` 访问容器：

```php test.php
Hello world, <?= $name ?>!

<?php dump($this->container->get(MyService::class)); ?>
```
