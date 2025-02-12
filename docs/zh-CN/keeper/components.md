# Keeper — 组件

## 数据网格

DataGrids 的 UI 组件提供了 [DataGrids](../component/data-grid.md) 的 UI 组件

它们位于 keeper 捆绑包中，可以这样引入：

```xhtml
<use:bundle path="keeper:bundle"/>
```

参见演示仓库中的 [使用示例](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/datagrids.dark.php)

### 使用组件

一个简单的数据网格声明看起来像这样：

```xhtml

<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:cell.text name="id" label="#"/>
    <grid:cell.text name="firstName" label="First Name"/>
    <grid:cell.text name="lastName" label="Last Name"/>
</ui:grid>
```

DataGrids 以声明式方式描述。代码不会在 php/stempler 阶段迭代，而是在 JavaScript 阶段迭代，这意味着您不能在声明中使用 php 条件进行单元格渲染。如果需要单元格的条件渲染器，则需要在 JavaScript 中编写一个。

列的定义应优先考虑语义而不是列名。通常，大多数列将与排序键匹配。例如，如果您有一行数据包含用户的名和姓，则可能希望有一个列 `name`，其中包含 `firstName` 和 `lastName` 连接起来。

```xhtml

<grid:cell.link name="name" label="Name" url="@action('users.edit', ['user' => '{id}'])" sort="true">
    {firstName}&nbsp;{lastName}
</grid:cell.link>
```

### 包装器 (ui:grid)

```xhtml

<ui:grid
        url="/some/url"
        method="GET"
        id="my-grid"
        namespace="foo"
        capture-forms="['form1','form2']"
        capture-filters="['filter1','filter2']"
        paginate-options="[10,20,30]"

        actions-title=""
        actions-label="Actions"
        actions-kind=""
        actions-icon="cog"
        actions-size="sm"
        actions-class=""
        actions-cell-class="">

</ui:grid>
```

| 参数           | 必需  | 默认值           | 描述                                                                                                                                                                                                                                                                                                                                                                      |
|-----------------|-------|-----------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| url             | 是    | -               | 实现 DataGrid API 的 URL                                                                                                                                                                                                                                                                                                                                                  |
| method          | 否    | GET             | 使用的 HTTP 方法，GET 或 POST                                                                                                                                                                                                                                                                                                                                                 |
| id              | 否    | [自动生成]      | 要使用的数据网格的 ID                                                                                                                                                                                                                                                                                                                                                         |
| namespace       | 否    | [空]            | 数据网格过滤器序列化中使用的字段名前缀。当页面上存在多个数据网格时使用。例如，如果 `namespace="foo"`，过滤器值 `bar=1` 将在 URL 中变为 "foo-bar=1"。如果页面上存在多个数据网格，则开发者有责任使用命名空间。否则行为是不可预测的。 |
| capture-forms   | 否    | [空]            | 字符串的 JSON 数组。它通过表单的 ID 将表单作为过滤器字段源附加到数据网格。它可以用于与数据网格在视觉上分离的过滤器，例如，在导航面板或侧边栏中。它在内部用于附加使用 `<ui:filter>` 定义的过滤器。                                                                                                                                                                     |
| capture-filters | 否    | [空]            | 附加 [过滤器切换按钮](https://github.com/spiral/toolkit/tree/master/packages/datagrid/src/filter-toggle) 的实例                                                                                                                                                                                                                                                               |
| paginate-options| 否    | [空]            | 默认分页器的选项                                                                                                                                                                                                                                                                                                                                                          |
| action-*        | 否    | [如示例所示]  | 用于生成“操作”按钮和相应列的一组参数                                                                                                                                                                                                                                                                                                                                               |

**限制**: 所有的过滤器表单都应该生成一个没有文件的扁平对象映射。目前不支持嵌套和/或数组。

### 过滤器 (grid:filter)

Grid:filter 组件直接将过滤器和/或搜索表单附加到网格。在内部，它使用“capture-forms”参数，该参数从技术上允许将任意数量的表单附加到数据网格。

```xhtml
<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:filter search="true" immediate="300" buttons="true">
        <form:input name="firstName" label="First Name" value="" size="6" required="true"/>
        <form:input name="lastName" label="Last Name" value="" size="6" required="true"/>
        <form:input name="email" label="Email" value="" required="true"/>
    </grid:filter>
</ui:grid>
```

| 参数      | 必需  | 默认值  | 描述                                                                                                                                                                                                                                                                                                                                             |
|-----------|-------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| search    | 否    | false    | 在右上角渲染一个搜索输入框                                                                                                                                                                                                                                                                                                          |
| search-name| 否    | "search" | 搜索字段的名称                                                                                                                                                                                                                                                                                                                            |
| fields    | 否    | -        | 字符串的 JSON 数组。如果您希望过滤器按钮正确地指示活动过滤器的数量，请将所有表单字段名称列为字符串的 JSON 数组。对于上面的示例，这将是 `fields="['firstName','lastName','email']"`。如果未指定，过滤器按钮将不会指示过滤器值是否处于活动状态。 |
| refresh   | 否    | false    | 渲染一个刷新按钮以触发数据刷新                                                                                                                                                                                                                                                                                                        |
| immediate | 否    | -        | 如果指定，搜索输入将没有“提交”按钮，并且搜索将在用户键入时执行，其去抖动值等同于此参数值（以毫秒为单位）。                                                                                                                                                                                                                              |
| [标签体]   | 否    | -        | 标签内部用作一个附加的过滤器表单，如果指定，将在模态窗口中显示                                                                                                                                                                                                                                                                    |
| buttons   | 否    | false    | 如果指定为 true，则默认的“清除”和“应用”按钮将附加到标签体                                                                                                                                                                                                                                                                                  |

请注意，过滤器表单是与“搜索输入”表单分开的表单，其重置按钮仅对过滤器表单有效。

#### 仅搜索

```xhtml
<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:filter search="true"/>
</ui:grid>
```

![Search Only](https://user-images.githubusercontent.com/16134699/103222727-b98c3500-4935-11eb-989a-801eb5649529.png)

#### 搜索和模态

```xhtml
<ui:grid url="@action('users.list', inject('params', []))" namespace="main">
    <grid:filter search="true" immediate="300" buttons="true">
        <form:input name="firstName" label="First Name" value="" size="6" required="true"/>
        <form:input name="lastName" label="Last Name" value="" size="6" required="true"/>
        <form:input name="email" label="Email" value="" required="true"/>
    </grid:filter>
</ui:grid>
```

![Search And Modal](https://user-images.githubusercontent.com/16134699/103222726-b8f39e80-4935-11eb-9c76-ec23e5cecd2c.png)

### 单元格类型

许多单元格允许使用行字段作为变量的模板。

使用的模板系统是 [handlebars](https://handlebarsjs.com/) 模板。通常，接受模板的字段具有预处理器，将 `{` 转换为 `{{`，因此如果要输出类似 `{{firstName}} {{lastName}}` 的内容，则需要将其作为 `{firstName} {lastName}` 传递

#### 文本 (grid:text)

```xhtml
<grid:cell.text
        name="user"
        label="User Name"
/>
```

直接在单元格中输出字段

| 参数     | 必需  | 默认值 | 描述                                                                                     |
|----------|-------|--------|------------------------------------------------------------------------------------------|
| name     | 是    | -      | 语义列名，与服务器数据中的字段匹配                                                                |
| label    | 是    | -      | 列标签                                                                                     |
| sort     | 否    | -      | 启用列的排序                                                                                |
| sort-default | 否 | -      | 提供“true”以使该列成为默认排序的列。 只有一列应具有此属性                                                    |
| sort-dir | 否    | 'asc'  | 默认排序方向，'asc' 或 'desc'                                                                 |

#### 链接 (grid:link)

```xhtml
<grid:cell.link
        name="user"
        title="Edit User {firstName}"
        body="{fistName}"
        href="/edit/{id}"
/>
```

输出一个链接

| 参数     | 必需  | 默认值 | 描述                                                                                                                                                                                                                                         |
|----------|-------|--------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| name     | 是    | -      | 语义列名，与服务器数据中的字段匹配。如果使用排序，则应与排序名称匹配。 内容是字段本身作为默认值，但可以使用“body”参数进行自定义以输出另一个字段                                                                                                                                         |
| label    | 是    | -      | 列标签                                                                                                                                                                                                                                         |
| href     | 是    | -      | 用于链接 URL 的模板                                                                                                                                                                                                                            |
| title    | 否    | -      | 用于链接标题文本的模板                                                                                                                                                                                                                          |
| body     | 否    | -      | 用于链接正文的模板。除了指定“body”属性之外，还可以使用标签体                                                                                                                                                                                          |
| sort     | 否    | -      | 启用列的排序                                                                                                                                                                                                                                    |
| sort-default | 否 | -      | 提供“true”以使该列成为默认排序的列。 只有一列应具有此属性                                                                                                                                                                                 |
| sort-dir | 否    | 'asc'  | 默认排序方向，'asc' 或 'desc'                                                                                                                                                                                                                    |

#### 日期 (grid:date)

```xhtml
<grid:cell.link
        name="created"
        label="Created At"
        format="LLL dd, yyyy HH:mm"
        sort="true"
        sort-dir="desc"
        sort-default="true"
/>
```

输出一个日期

| 参数     | 必需  | 默认值            | 描述                                                                                                                   |
|----------|-------|-------------------|------------------------------------------------------------------------------------------------------------------------|
| name     | 是    | -                 | 语义列名，与服务器数据中的字段匹配。它接受带有时区的 ISO 日期。                                                              |
| label    | 是    | -                 | 列标签                                                                                                                 |
| format   | 否    | LLL dd, yyyy HH:mm| 日期格式。 参见 [Luxon 文档以获取可用令牌](https://moment.github.io/luxon/docs/manual/parsing.html#table-of-tokens)       |
| sort     | 否    | -                 | 启用列的排序                                                                                                             |
| sort-default | 否 | -                 | 提供“true”以使该列成为默认排序的列。 只有一列应具有此属性。                                                               |
| sort-dir | 否    | 'asc'             | 默认排序方向，'asc' 或 'desc'                                                                                              |

#### 动作

提供的一种单独的标签是动作标签。使用它们中的任何一个都会添加“actions”列，该列带有一个带有动作下拉列表的按钮。每个动作的标签都会将一个动作附加到该下拉列表中。

要自定义按钮的外观，请使用 `ui:grid` 的 `action-*` 参数

![Actions](https://user-images.githubusercontent.com/16134699/103222723-b85b0800-4935-11eb-8683-87e3cbcfd28a.png)

##### 动作：链接

```xhtml
<grid:action.link
        href="edit/{id}"
        template="{id}"
        label="Edit"
        title="Edit {firstName}"
        icon="edit"
        target="_blank"
/>
```

输出一个链接

| 参数    | 必需  | 默认值 | 描述                                                                                                                                                         |
|---------|-------|--------|--------------------------------------------------------------------------------------------------------------------------------------------------------------|
| href    | 是    | -      | 用于链接 URL 的模板                                                                                                                                            |
| title   | 否    | -      | 用于链接标题文本的模板                                                                                                                                          |
| target  | 否    | _self  | 指定链接的目标                                                                                                                                                   |
| template| 否    | -      | 用于链接正文的模板。除了指定“template”属性之外，还可以使用标签体                                                                                                                                    |
| label   | 否    | -      | 如果未使用模板参数，则用于链接文本的文本。如果作为未转义的值传递，则支持模板，例如 `label = "{!! '{{variable}}' !!}"`                                                                                          |
| icon    | 否    | -      | 如果未使用模板参数，则用于链接图标的名称。它使用 Font Awesome 图标。如果作为未转义的值传递，则支持模板，例如 `icon = "{!! '{{variable}}' !!}"`                                                                        |

##### 动作：动作

```xhtml
<grid:action.action
        href="edit/{id}"
        template="{id}"
        label="Edit"
        title="Edit {firstName}"
        icon="edit"
        method="POST"
        data="{ foo: 1}"
        refresh="true"
        confirm="true"
        confirm-title="true"
        confirm-ok="true"
        confirm-cancel="true"
/>
```

向服务器发出请求

| 参数         | 必需  | 默认值 | 描述                                                                                                                                                                                                                                                  |
|--------------|-------|--------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| href         | 是    | -      | 用于动作 URL 的模板                                                                                                                                                                                                                                       |
| method       | 否    | GET    | 要使用的方法名称                                                                                                                                                                                                                                         |
| title        | 否    | -      | 用于动作标题文本的模板                                                                                                                                                                                                                                     |
| template     | 否    | -      | 用于链接正文的模板。除了指定“template”属性之外，还可以使用标签体                                                                                                                                                                                          |
| label        | 否    | -      | 如果未使用模板参数，则用于链接文本的文本。如果作为未转义的值传递，则支持模板，例如 `label = "{!! '{{variable}}' !!}"`                                                                                                                                                         |
| icon         | 否    | -      | 如果未使用模板参数，则用于链接图标的名称。它使用 Font Awesome 图标。如果作为未转义的值传递，则支持模板，例如 `icon = "{!! '{{variable}}' !!}"`                                                                                                                          |
| body         | 否    | -      | 要作为请求正文附加的 JSON 字符串。如果作为未转义的值传递，则支持模板，例如 `data = "{!! '{ "id": {{variable}} }' !!}"`                                                                                                                                                              |
| refresh      | 否    | -      | 一个布尔值，指示来自服务器的成功响应是否应触发数据网格刷新                                                                                                                                                                                                          |
| condition    | 否    | -      | 包含布尔变量的字段名称，指示是否应显示该动作。或者，可能有一个 handlebars 模板，如果需要更复杂的逻辑，该模板应为 false 值返回 ''，或者为正值返回任何其他内容                                                                                                                          |
| toastError   | 否    | -      | 错误发生时显示的 Toast 消息的模板                                                                                                                                                                                                                           |
| toastSuccess | 否    | -      | 动作成功时显示的 Toast 消息的模板                                                                                                                                                                                                                          |
| confirm      | 否    | -      | 如果指定，则指定 confirm-title、confirm-ok 和 confirm-cancel。它将在执行动作之前显示确认对话框。`confirm` 指定确认对话框正文的 handlebars 模板                                                                                                                                                                     |
| confirm-title| 否    | -      | 指定确认对话框标题的 handlebars 模板                                                                                                                                                                                                                          |
| confirm-ok   | 否    | -      | 指定确认对话框的正结果按钮文本的 handlebars 模板                                                                                                                                                                                                                        |
| confirm-cancel| 否    | -      | 指定确认对话框的负结果按钮文本的 handlebars 模板                                                                                                                                                                                                                        |

##### 动作：删除

```xhtml
<grid:action.delete
        href="edit/{id}"
/>
```

它与“grid:action”相同，但具有预定义的确认文本和危险类。通常只需要该操作类型的 `href`。

#### 模板 (grid:template)

```xhtml
<grid:cell.template
        name="user"
        label="User Name"
        body="<i class='fa fa-user'></i> {firstName} {lastName}"
/>
```

输出自定义模板

| 参数     | 必需  | 默认值 | 描述                                                                                                                                                                                                                                              |
|----------|-------|--------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| name     | 是    | -      | 语义列名，与服务器数据中的字段匹配。如果使用排序，则应与排序名称匹配。内容是字段本身作为默认值，但可以使用“body”参数进行自定义以输出另一个字段                                                                                                                                                  |
| label    | 是    | -      | 列标签                                                                                                                                                                                                                                              |
| body     | 否    | -      | 用于正文的模板。除了指定“body”属性之外，还可以使用标签体                                                                                                                                                                                              |
| sort     | 否    | -      | 启用列的排序                                                                                                                                                                                                                                        |
| sort-default | 否 | -      | 提供“true”以使该列成为默认排序的列。 只有一列应具有此属性                                                                                                                                                                                  |
| sort-dir | 否    | 'asc'  | 默认排序方向，'asc' 或 'desc'                                                                                                                                                                                                                         |

#### 完全自定义渲染 (grid:render)

```xhtml
<grid:cell.render
        name="roles"
        label="Roles"
        renderer="roles"
/>
```

它使用自定义函数来渲染单元格

| 参数     | 必需  | 默认值 | 描述                                                                                                                                                                                                                                       |
|----------|-------|--------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| name     | 是    | -      | 与服务器数据中的字段匹配的语义列名。如果使用排序，则应与排序名称匹配。内容是字段本身作为默认值，但可以使用“renderer”参数进行自定义以输出另一个字段                                                                                                                                             |
| label    | 是    | -      | 列标签                                                                                                                                                                                                                                       |
| renderer | 否    | -      | 一个预先注册的函数名称，该函数能够渲染单元格                                                                                                                                                                                                                   |
| sort     | 否    | -      | 启用列的排序                                                                                                                                                                                                                                   |
| sort-default | 否 | -      | 提供“true”以使该列成为默认排序的列。 只有一列应具有此属性                                                                                                                                                                                  |
| sort-dir | 否    | 'asc'  | 默认排序方向，'asc' 或 'desc'                                                                                                                                                                                                                    |

要为数据网格定义自定义函数，请在工具包声明之前添加一个 script 标签。如果您看到“找不到渲染器”错误，则很可能您在代码中太晚输出了渲染器。

```xhtml
<stack:push name="scripts" unique-id="datagrid-roles-renderer">
    <script type="text/javascript">
        window.SFToolkit_tools_datagrid = window.SFToolkit_tools_datagrid || {};
        window.SFToolkit_tools_datagrid['roles'] = function () {
            return function (roles) {
                return roles.map(function (role) {
                    return '<span class="badge badge-primary mr-1">' + role.toUpperCase() + '</span>'
                }).join('');
            }
        };
    </script>
</stack:push>
```

render 函数返回的函数应该是
[CellRenderFunction](https://github.com/spiral/toolkit/blob/master/packages/datagrid/src/types.ts#L79)，它接受许多有用的参数，允许以非常灵活的方式自定义渲染。

### 高级用法

DataGrid 基于 JavaScript 库，可以使用 JavaScript 初始化

JavaScript 声明允许更大的灵活性，例如，您可以拥有多个动作列、批量动作和选择、代码中直接的自定义渲染器，而无需预先定义它们。

有关更多详细信息，请参阅 [DataGrid 源代码](https://github.com/spiral/toolkit/tree/master/packages/datagrid/src)。

![Custom](https://user-images.githubusercontent.com/16134699/103222725-b8f39e80-4935-11eb-8b35-464ae0224906.png)
![Custom](https://user-images.githubusercontent.com/16134699/103222724-b85b0800-4935-11eb-9c2f-5c4cd3eaa051.png)

```xhtml
<div class="sf-table">
    <div class="js-sf-datagrid">
        @declare(syntax=off)
        <script type="text/javascript" role="sf-options">
            (function () {
                return {
                    "id": "custom",
                    "url": "/keeper/users/list",
                    "namespace": "custom",
                    "method": "GET",
                    "ui": {
                        "headerCellClassName": {"actions": "text-right"},
                        "cellClassName": {"actions": "text-right py-2", "created": "text-nowrap"}
                    },
                    "paginator": {"limitOptions": [10, 20, 50, 100]},
                    "sort": "created",
                    "columns": [{"id": "name", "title": "Name", "sortDir": "asc"}, {"id": "actions2", "title": " "}, {
                        "id": "email",
                        "title": "Email",
                        "sortDir": "asc"
                    }, {"id": "created", "title": "Created At", "sortDir": "desc"}, {
                        "id": "roles",
                        "title": "Roles",
                        "sortDir": null
                    }, {"id": "id", "title": "ID", "sortDir": null}, {"id": "actions", "title": " "}],
                    "selectable": {
                        "type": "multiple",
                        "id": "id"
                    },
                    "renderers": {
                        "cells": {
                            "name": {
                                "name": "link",
                                "arguments": {
                                    "title": "",
                                    "body": "{{firstName}}&nbsp;{{lastName}}",
                                    "href": "\/keeper\/users\/{{id}}"
                                }
                            },
                            "email": {
                                "name": "link",
                                "arguments": {"title": "", "body": "{{email}}", "href": "mailto:{{email}}"}
                            },
                            "created": {"name": "dateFormat", "arguments": ["LLL dd, yyyy HH:mm"]},
                            "roles": {"name": "roles", "arguments": []},
                            "id": {"name": "template", "arguments": ["{{id}}"]},
                            "actions": {
                                "name": "actions",
                                "arguments": {
                                    "kind": "",
                                    "size": "sm",
                                    "className": "",
                                    "icon": "cog",
                                    "label": "Actions",
                                    "actions": [{
                                        "type": "href",
                                        "url": "\/keeper\/users\/{{id}}",
                                        "label": "Edit",
                                        "target": null,
                                        "icon": "edit",
                                        "template": ""
                                    }, {
                                        "type": "action",
                                        "url": "\/keeper\/users\/{{id}}",
                                        "method": "DELETE",
                                        "label": "Delete",
                                        "icon": "trash",
                                        "template": "<span class=\"text-danger\"><i class=\"fa fw fa-trash\"><\/i>&nbsp;&nbsp; Delete<\/span>",
                                        "condition": null,
                                        "data": [],
                                        "refresh": true,
                                        "confirm": {
                                            "body": "Are you sure to delete this entry?",
                                            "title": "Confirmation Required",
                                            "confirm": "Delete",
                                            "confirmKind": "danger",
                                            "cancel": "Cancel"
                                        },
                                        "toastSuccess": "<i class=\"fa fa-check-circle\"><\/i>&nbsp; {{message}}\n              ",
                                        "toastError": "<i class=\"fa fa-exclamation\"><\/i>&nbsp; {{error}}\n              "
                                    }]
                                }
                            },
                            "actions2": {
                                "name": "actions",
                                "arguments": {
                                    "kind": "",
                                    "size": "sm",
                                    "className": "",
                                    "icon": "cog",
                                    "label": "Actions 2",
                                    "actions": [{
                                        "type": "href",
                                        "url": "\/keeper\/users\/{{id}}",
                                        "label": "Edit",
                                        "target": null,
                                        "icon": "edit",
                                        "template": ""
                                    }, {
                                        "type": "action",
                                        "url": "\/keeper\/users\/{{id}}",
                                        "method": "DELETE",
                                        "label": "Delete",
                                        "icon": "trash",
                                        "template": "<span class=\"text-danger\"><i class=\"fa fw fa-trash\"><\/i>&nbsp;&nbsp; Delete<\/span>",
                                        "condition": null,
                                        "data": [],
                                        "refresh": true,
                                        "confirm": {
                                            "body": "Are you sure to delete this entry?",
                                            "title": "Confirmation Required",
                                            "confirm": "Delete",
                                            "confirmKind": "danger",
                                            "cancel": "Cancel"
                                        },
                                        "toastSuccess": "<i class=\"fa fa-check-circle\"><\/i>&nbsp; {{message}}\n              ",
                                        "toastError": "<i class=\"fa fa-exclamation\"><\/i>&nbsp; {{error}}\n              "
                                    }]
                                }
                            }
                        },
                        "actions": {
                            "delete": {
                                renderAs: "<div class='btn btn-danger'>Delete</div>",
                                onClick: function (state, grid) {
                                    console.log(state, grid);
                                }
                            }
                        }
                    }
                };
            });
        </script>
        @declare(syntax=on)
    </div>
</div>
```

## 表单

用于表单的 UI 组件

它们位于工具包捆绑包中，可以这样引入：

```xhtml
<use:bundle path="toolkit:bundle"/>
```

当使用时，这也会自动包含

```xhtml
<use:bundle path="keeper:bundle"/>
```

参见演示仓库中的 [使用示例](https://github.com/spiral/app-keeper/blob/master/app/views/keeper/showcase/forms.dark.php)

### 用法

表单主要是常规的 HTML 表单，它们在其之上具有 ajax 提交功能

```xhtml
<form:wrapper action="/" method="PUT">
    <form:input name="firstName" label="First Name" value="" size="6" required="true"/>
    <form:input name="lastName" label="Last Name" value="" size="6" required="true"/>
    <form:input name="email" label="Email" value="" required="true"/>

    <form:input type="password" name="password" label="New Password" size="6" required="true"/>
    <form:input type="password" name="confirmPassword" label="Confirm Password" size="6"/>

    <form:label label="User Roles" name="roles" required="true">
        @foreach(['admin'=>'Admin', 'super-admin'=>'Super Admin'] as $role => $label)
        <form:checkbox id="role-{{ $role }}" name="roles[]" value="{{ $role }}" label="{{$label}}"/>
        @endforeach
    </form:label>

    <form:select
            label="Select Something"
            values="{{ [1 => 'First', 2 => 'Second', 3 => 'Third'] }}"
            value="2"
            placeholder="Select Value"
    />

    <form:radio-group
            name="radios"
            values="{{ [1 => 'First', 2 => 'Second', 3 => 'Third'] }}"
            value="2"
    />

    <form:button label="Create"/>
</form:wrapper>
```

![Form image](https://user-images.githubusercontent.com/16134699/103222722-b85b0800-4935-11eb-9cb1-45a8c44f1834.png)

#### form:wrapper

| 参数          | 必需  | 默认值 | 描述                                                                                                                                                                         |
|---------------|-------|--------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| action        | 是    | -      | 表单的 action URL                                                                                                                                                               |
| method        | 否    | POST   | 要使用的 Http 方法，GET 或 POST                                                                                                                                                    |
| id            | 否    | -      | 表单的 ID（如果希望对其进行硬编码）                                                                                                                                               |
| class         | 否    | -      | 要添加到包装器的类                                                                                                                                                               |
| immediate     | 否    | -      | 去抖动值（以毫秒为单位）。如果指定，则任何更改事件都将触发表单提交                                                                                                                             |
| submit-on-reset| 否    | false  | 提交“重置”事件（即重置按钮）上的表单。 对数据网格的过滤器很有用                                                                                                                             |
| data-before-submit | 否 | -      | 全局 JS 作用域中将在提交表单之前调用的回调函数的名称。如果该回调返回 false，则不会提交表单。                                                                                                            |
| data-after-submit | 否 | -      | 全局 JS 作用域中将在提交表单后调用的回调函数的名称。请注意，您希望自己检查提交表单的结果。 |

如果您需要对表单及其回调进行更细粒度的控制，请参考
[源代码](https://github.com/spiral/toolkit/blob/master/packages/form/src/Form.ts)

#### form:*

大多数表单输入都共享此处列出的通用属性

| 参数     | 必需  | 默认值 | 描述                           |
|----------|-------|--------|--------------------------------|
| label    | 否    | -      | 在输入之前呈现的标签              |
| required | 否    | false  | 如果为 true，则在标签附近呈现红色的 `*` |
| wrapper-id| 否    | -      | 用于包装器 div 的 ID           |
| wrapper-class| 否    | -      | 要添加到包装器的类              |
| size     | 否    | 12     | 网格系统的列大小                  |
| error    | 否    | -      | 预渲染的错误反馈文本             |
| success  | 否    | -      | 预渲染的成功反馈文本             |
| help     | 否    | -      | 描述文本                        |

**重要提示**: 大多数输入字段会将所有未识别的属性代理到内部的相应输入。

#### form:input

简单的表单输入

| 参数      | 必需  | 默认值 | 描述                              |
|-----------|-------|--------|---------------------------------|
| name      | 是    | -      | 字段名称                           |
| value     | 否    | -      | 字段值                           |
| placeholder| 否    | -      | 字段占位符                        |
| class     | 否    | -      | 要为字段渲染的附加类              |
| type      | 否    | -      | 字段类型属性                       |
| disabled  | 否    | -      | 如果应禁用该字段                  |

```xhtml
<form:input name="firstName" label="First Name" value="" size="6" required="true"/>
```

![input](https://user-images.githubusercontent.com/16134699/103222711-b6914480-4935-11eb-8bc7-a30dbacc9eaf.png)

#### form:textarea

简单的表单文本区域

| 参数      | 必需  | 默认值 | 描述                              |
|-----------|-------|--------|---------------------------------|
| name      | 是    | -      | 字段名称