# Keeper — 视图

Keeper 框架附带若干 stempler 指令、视图和模板

## 指令

### 权限验证指令

提供带有上下文的权限检查，并在底层使用 `GuardInterface` 调用。

```html
@auth('permision', ['some' => 'context'])
<p>[[Allowed]]</p>
@endauth
```

### 退出指令

包装给定的退出 URL 参数，并添加当前的认证令牌：

```html
<a href="@logout(admin['auth:logout'])">[[Log out]]</a>
```

目前，该指令接受完整的路由名称。

### 路由指令

有一个方便的指令用于在给定的命名空间中生成 URI：

```html
<a href="@keeper('admin', 'createUser')">[[+ User]]</a>
```

## 视图和模板

接下来要提到的是以下视图和模板：

- `keeper:login` 视图包含注册表单，可以进行扩展以进行一些自定义，或者从头开始构建（不要忘记在 keeper 配置中注册该视图）：

```php
<?php
// app/config/path/to/keeper/config.php

return [
    'loginView' => 'default:path/to/login/view'
];
```

- `keeper:layout/page` 和 `keeper:layout:tabs` 是两个主要的布局，示例如下：

```html

<extends:keeper:layout.page title="[[Title]]"/>
<use:bundle path="keeper:bundle"/>

<define:content>
    [[Some page content.]]
</define:content>
```

```html

<extends:keeper:layout.tabs title="[[Title]]"/>
<use:bundle path="keeper:bundle"/>

<ui:tab id="information" icon="info" title="[[Information]]" active="true">
    [[Information tab content.]]
</ui:tab>
<ui:tab id="data" icon="cog" title="[[Data]]">
    [[Data tab content.]]
</ui:tab>
```

- 网格模板是功能强大的 HTML 元素，用于创建表格：

```html
//...
<block:content>
    <ui:grid url="@action('users.list')">
        <grid:filter search="true" immediate="300" buttons="true">
            @auth('keeper.permision')
            <form:select name="status" label="[[Status]]" placeholder="[[Select Status]]"
                         values="{{ ['active' => '[[Active]]', 'disabled' => '[[Disabled]]'] }}"/>
            @endauth
        </grid:filter>

        <grid:cell.text name="first_name" label="[[First Name]]" sort="true" body="{firstName}" sort-dir="asc"
                        sort-default="true"/>
        <grid:cell.text name="last_name" label="[[Last Name]]" sort="true" body="{lastName}"/>
        <grid:cell.link name="email" label="[[Email]]" href="mailto:{email}" body="{email}" sort="true"
                        condition="showEmail"/>
        <grid:cell.render name="status" label="[[Status]]" renderer="status"/>

        <grid:action.link label="[[Edit]]" icon="edit" url="@action('users.edit', ['user' => '{id}'])"
                          permission="keeper.users.view"/>
    </ui:grid>
</block:content>
<stack:push name="scripts" unique-id="datagrid-account-renderer">
    <script type="text/javascript">
        SFToolkit.tools._datagrid.register('status', function () {
            return function (status, item) {
                let map = {
                    "active": 'badge-primary',
                    "disabled": 'badge-warning',
                }
                let badge = (status.toLowerCase() in map) ? map[status.toLowerCase()] : 'badge-secondary';
                return '<span class="badge ' + badge + ' mr-1">' + status.toUpperCase() + '</span>';
            }
        });
    </script>
</stack:push>
```

#### 解释说明：

`sort="true"` 允许按该列进行排序。
额外的 `sort-dir="asc|desc"` 和 `sort-default="true"` 启用默认排序。
`name="first_name"` 被用作查询中排序键的名称 (`http://example.com?sort[first_name]=asc`)。

`condition=""` 允许根据需要显示/隐藏行。`body="{firstName}"` 是行中列的名称。

`<grid:cell.render />` 具有 `renderer` 属性，允许使用自定义渲染模板。当前项目（网格行）和单元格值在内部可用。

网格示例：

```json
{
  "status": 200,
  "data": [
    {
      "firstName": "John",
      "lastName": "Smith",
      "email": "john@smith.com",
      "showEmail": true,
      "status": "active"
    }
  ]
}
```
