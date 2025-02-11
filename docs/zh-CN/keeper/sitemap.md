# Keeper — 站点地图

站点地图是一个用于构建菜单导航和面包屑的子模块。它由 4 种类型的节点组成：

- 链接 (link) 和视图 (view) 代表一个页面
- 片段 (segment) 和组 (group) 是嵌套元素的容器。

站点地图可以通过注解 (annotations) 或直接使用 `Sitemap` 模块进行声明。

## 直接声明站点地图

创建一个自定义的引导程序 (bootloader)，注入 `Sitemap` 并将其添加到 `KeeperBootloader` 依赖项中：

```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader;

class KeeperBootloader extends Bootloader\KeeperBootloader
{
    protected const LOAD = [
        Bootloader\SitemapBootloader::class,
        NavigationBootloader::class,
    ];
}
```

现在你就可以声明站点地图节点了：

```php
<?php

declare(strict_types=1);

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends Bootloader
{
    public function boot(Sitemap $sitemap): void
    {
        $sitemap->link('dashboard.index', 'Dashboard', ['icon' => 'home']);
        // ...
    }
}
```

> **注意**
> 在这种情况下，你将会难以使用站点地图注解。这种方法适用于你不想使用注解的情况。

我们推荐扩展 `SitemapBootloader` 并使用 `declareSitemap()` 方法来声明结构：

```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        $sitemap->link('dashboard.index', 'Dashboard', ['icon' => 'home']);

        $group = $sitemap->group('users', 'Users and Groups', ['icon' => '...']);
        if ($group !== null) {
            $users = $group->link('users.index', 'Users', ['icon' => '...']);
            if ($users !== null) {
                $users->view('users.create', 'Create User');
                $users->view('users.edit', 'Edit User');
            }
            $group->link('groups.index', 'Groups', ['icon' => '...']);
        }
        // ...
    }
}
```

> **注意**
> 在这种情况下，站点地图模块将能够与站点地图属性 (attributes) 同步。

## 属性 (Attributes)

`Group` 和 `Segment` 注解可以应用于类，而 `Link` 和 `View` 则用于类方法。
`Link` 和 `View` 只有当方法也有 `Action` 注解时才能起作用。

```php
<?php

declare(strict_types=1);

namespace App\Controller\Keeper;

use Spiral\Keeper\Annotation as Keeper;
use Spiral\Views\ViewsInterface;

 #[Keeper\Controller(name: "users", prefix: "/users")]
 #[Keeper\Sitemap\Group(name: "users", title: "Users and Groups")]
class UsersController
{
     #[Keeper\Action(route: "", methods: "GET")]
     #[Keeper\Sitemap\Link(title: "Users", options: ["icon" => "user-friends"])]
    public function index(ViewsInterface $views): string
    {
        return $views->render('keeper:users/list');
    }

     #[Keeper\Action(route: "/create", methods: "GET")]
     #[Keeper\Sitemap\View(title: "Add new user")]
    public function new(ViewsInterface $views): string
    {
        return $views->render('keeper:users/create');
    }
    // ...
}
```

在引导程序中声明组和片段 (segments) 及其完整信息，并在属性中仅使用名称 (在单个组下有多个控制器的情况下) 比较方便。请看下面的例子：

```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        $sitemap->group('users', 'Users and Groups', ['icon' => '...']);
        // ...
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Controller\Keeper;

use Spiral\Keeper\Annotation as Keeper;
use Spiral\Views\ViewsInterface;

 #[Keeper\Controller(name: "users", prefix: "/users")]
 #[Keeper\Sitemap\Group(name: "users")]
class UsersController
{
    // ...
}
```

此外，如果你的嵌套结构很复杂，你可以在引导程序和控制器中声明它，然后仅使用最接近的元素：

```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        $sitemap
            ->group('one', 'SuperGroup')
            ->segment('two', 'SuperSegment')
            ->group('three', 'FinalGroup');
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Controller\Keeper;

use Spiral\Keeper\Annotation as Keeper;
use Spiral\Views\ViewsInterface;

 #[Keeper\Controller(name: "users", prefix: "/users")]
 #[Keeper\Sitemap\Group(name: "three")]
class UsersController
{
    // ...
}
```

## 可见性 (Visibility)

默认情况下，所有节点对用户都是可见的，`withVisibleNodes()` 允许隐藏禁止的节点。如果任何节点被禁止，它将从树中与它的所有子节点一起被移除。此外，传递 `$targeNode` 将会标记所有活动节点（如果找到匹配项），因此它将允许你使用面包屑。

权限从 `#[GuardNamespace]`，`#[Guarded]` 和 `#[Link]` 属性中获取：
`<guard Namespace (or controller name)>.<link permission (or guarded permission (or method name))>`.

在方法受到基于上下文的权限规则保护的情况下，使用 `#[Link]` 权限 — 为了在侧边栏和面包屑中渲染链接，不能传递上下文，所以你必须使用额外的导航权限（并使用允许规则注册它）。在其他情况下，你可以依赖标准的 `#[Guarded]` 权限（或方法名称）流程。

> **注意**
> 请注意，这里的 keeper 命名空间 (namespace) 没有被自动使用，因为这些注解来自外部模块。

`#[GuardNamespace]` 属性的示例：

```php
 #[Controller(name: "with", prefix: "/with", namespace: "first")]
 #[GuardNamespace(namespace: "withNamespace")]
class WithNamespaceController
{
     #[Link(title: "A")]
    public function a(): void
    {
        // permission is "withNamespace.a"
    }

     #[Link(title: "B")]
     #[Guarded(permission: "permission")]
    public function b(): void
    {
        // permission is "withNamespace.permission"
    }

     #[Link(title: "B", permission: "methodC")]
     #[Guarded(permission: "permission")]
    public function с(): void
    {
        // permission is "withNamespace.methodC"
    }
}
```

没有 `#[GuardNamespace]` 属性的示例：

```php
#[Controller(name: "without", prefix: "/without", namespace: "second")]
class WithoutNamespaceController
{
    #[Link(title: "A")]
    public function a(): void
    {
        // permission is "without.a"
    }

     #[Link(title: "B")]
     #[Guarded(permission: "permission")]
    public function b(): void
    {
        // permission is "without.permission"
    }

     #[Link(title: "C", permission: "methodC")]
     #[Guarded(permission: "permission")]
    public function с(): void
    {
        // permission is "without.methodC"
    }
}
```

## 排序 (Sorting)

默认情况下，站点地图会按照它们在类中被找到或直接在 `Sitemap` 中声明的方式对节点进行排序。你可以在注解中使用 `position`（浮点数）属性，或者在直接声明选项中使用 `position` 选项。节点将按照升序进行排序（如果位置匹配，则按出现顺序排序）。

```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        $sitemap->group('users', 'Users and Groups', ['position' => -0.8]);
    }
}
```

```php
<?php

declare(strict_types=1);

namespace App\Controller\Keeper;

use Spiral\Keeper\Annotation as Keeper;
use Spiral\Views\ViewsInterface;

 #[Keeper\Controller(name: "users", prefix:"/users")]
 #[Keeper\Sitemap\Group(name: "users")]
class UsersController
{
     #[Keeper\Action(route: "", methods: "GET")]
     #[Keeper\Sitemap\Link(title: "Users", position: 2.7)]
    public function index(ViewsInterface $views): string
    {
        return $views->render('keeper:users/list');
    }
}
```

## 嵌套 (Nesting)

任何节点都可以是另一个父节点的子节点。在属性中使用 `parent` 参数。
如果你指的是同一个控制器内的某个方法，则可以使用仅使用名称，否则使用 `controller.method` 符号：

```php
<?php

declare(strict_types=1);

namespace App\Controller\Keeper;

use Spiral\Keeper\Annotation as Keeper;
use Spiral\Views\ViewsInterface;

#[Keeper\Controller(name: "users", prefix: "/users")]
class UsersController
{
     #[Keeper\Action(route: "", methods: "GET")]
     #[Keeper\Sitemap\Link(title: "Users")]
    public function index(ViewsInterface $views): string
    {
        return $views->render('keeper:users/list');
    }

     #[Keeper\Action(route: "/create", methods: "GET")]
     #[Keeper\Sitemap\View(title: "Add new user", parent: "index")]
    public function new(ViewsInterface $views): string
    {
        return $views->render('keeper:users/create');
    }

     #[Keeper\Action(route: "/create", methods: "GET")]
     #[Keeper\Sitemap\View(title: "Add new user", parent: "groups.index")]
    public function create(ViewsInterface $views): string
    {
        return $views->render('keeper:users/create');
    }
    // ...
}
```

对于直接的站点地图声明，你在声明后会收到一个新的节点，因此你可以使用链式调用：

```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        // [root]->[link]
        $sitemap->link('dashboard.index', 'Dashboard', ['icon' => 'home']);

        $group = $sitemap->group('users', 'Users and Groups', ['icon' => '...']);
        if ($group !== null) {
            // [root]->[users group]
            $users = $group->link('users.index', 'Users', ['icon' => '...']);
            if ($users !== null) {
                // [root]->[users group]->[view]
                $users->view('users.create', 'Create User');
                $users->view('users.edit', 'Edit User');
            }
            // [root]->[users group]
            $group->link('groups.index', 'Groups', ['icon' => '...']);
        }
        // ...
    }
}
```

## 侧边栏 (Sidebar)

侧边栏不渲染 `View` 节点。可能的嵌套层次结构可以是：

- segment
    - group
        - link
    - link
- group
    - link
- link

> **注意**
> 即使站点地图支持任何类型的嵌套和深度，任何其他组合都将被忽略。

你可以使用 `icon` 选项为侧边栏添加图标。

## 面包屑 (Breadcrumbs)

与侧边栏相反，面包屑能够在各种嵌套和深度内渲染所有节点类型。

```php
<?php

declare(strict_types=1);

use Spiral\Keeper\Bootloader\SitemapBootloader;
use Spiral\Keeper\Module\Sitemap;

class NavigationBootloader extends SitemapBootloader
{
    public function declareSitemap(Sitemap $sitemap): void
    {
        $sitemap
            ->link('dashboard.index', 'Level 1')
            ->link('users.index', 'Level 2')
            ->link('groups.index', 'Level 3')
            ->link('logs.index', 'Level 4')
            ->link('system.index', 'Level 5');
    }
}
```

在 `system.index` 终点，面包屑将是（`[root]` 并且当前节点将被忽略）：

```
[dashboard.index]->[users.index]->[groups.index]->[logs.index]
```
