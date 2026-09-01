# Daiying CMS V1.2 主题开发规范

版本：2026-08-31  
适用版本：Daiying CMS V1.2

## 1. 基本原则

主题只负责前台展示：HTML、CSS、响应式、模板结构、少量前台交互和主题设置读取。主题不得修改业务数据，不得实现支付、库存、订单、授权、发卡、云存储等功能规则。

主题不得修改 `system/core`、`config`、`storage`、`public/index.php`，不得要求站长手动改 Core。

## 2. 主题目录

主题安装目录：

```text
content/themes/{theme_id}/
```

最小结构：

```text
content/themes/mytheme/
  theme.json
  templates/
    home.php
    list.php
    content.php
    error.php
  assets/
```

`theme_id` 必须与目录名一致。

## 3. theme.json

```json
{
  "theme_id": "mytheme",
  "name": "My Theme",
  "version": "1.0.0",
  "author": "Your Name",
  "core": {"min": "1.2.3", "max": "1.x"},
  "content_types": ["article", "page"],
  "recommended_plugins": [],
  "required_plugins": [],
  "settings_schema": {}
}
```

`theme_id` 规则：

- 小写字母开头。
- 只允许小写字母、数字、下划线。
- 长度 3 到 64。
- 禁止使用 `safe`。

## 4. settings_schema

推荐字段类型：

```text
text
textarea
url
email
color
toggle
range
image
```

常用属性：

| 属性 | 说明 |
| --- | --- |
| `type` | 字段类型 |
| `label` | 后台显示名 |
| `group` | 后台分组 |
| `order` | 排序 |
| `default` | 默认值 |
| `placeholder` | 占位提示 |
| `min` / `max` / `step` | 数值约束 |

图片字段要求：

- 如果主题自带官方 Logo，应默认使用主题资源，不应强迫用户填写 URL。
- `image` 设置保存值可能是字符串 URL，也可能是媒体对象。
- 主题读取时应兼容 `/uploads/...`、`uploads/...`、`content/uploads/...`、`http://...`、`https://...`。
- 输出前必须过滤 URL，禁止 `javascript:`、控制字符、引号和 HTML 注入。

## 5. 模板

推荐模板：

| 模板 | 用途 |
| --- | --- |
| `home.php` | 首页 |
| `list.php` | 列表页 |
| `content.php` | 文章和页面详情 |
| `error.php` | 404 或错误页 |
| `_theme.php` | 公共函数，可选 |

模板头部：

```php
<?php

declare(strict_types=1);

use Cms\Core\Theme\TemplateContext;

/** @var TemplateContext $context */
```

## 6. TemplateContext

常用方法：

```php
$context->get('title', '');
$context->setting('primary_color', '#1f6feb');
$context->e($value);
```

所有不可信文本必须使用 `$context->e()` 输出。

## 7. 内容上下文

`home.php` 常用：

| Key | 说明 |
| --- | --- |
| `site_name` | 站点名称 |
| `navigation` | 导航数组 |
| `contents` | 最新内容列表 |
| `seo` | SEO 数据 |

`list.php` 常用：

| Key | 说明 |
| --- | --- |
| `title` | 列表标题 |
| `items` | 内容列表 |
| `pagination` | 分页数据 |
| `term` | 分类或标签 |
| `empty_message` | 空列表提示 |

`content.php` 常用：

| Key | 说明 |
| --- | --- |
| `content` | 内容原始数据 |
| `title` | 标题 |
| `media` | 媒体 ViewModel |
| `rendered_blocks` | 已渲染区块 HTML |
| `paid_content` | 付费内容状态 |
| `categories` | 分类 |
| `tags` | 标签 |
| `comments` | 评论数据 |

内容正文推荐：

```php
<article><?= $context->get('rendered_blocks', '') ?></article>
```

不要自行解析 `blocks_json` 代替 Core 渲染。

## 8. XSS 转义

必须转义：

- 标题、摘要、正文外的字段。
- 导航名称、分类名、标签名。
- 主题设置文本。
- 表单值和错误提示。

URL、颜色和 CSS 值必须白名单校验。

## 9. 主题 ZIP

ZIP 必须包含单独根目录：

```text
mytheme/
  theme.json
  templates/home.php
```

限制：

- ZIP 最大约 10 MB。
- 解压后最大约 25 MB。
- 文件数不超过 500。
- 目录深度不超过 8。
- 不允许路径穿越、绝对路径、软链接、特殊文件、嵌套 ZIP。
- 不允许高风险可执行文件。

## 10. 兼容版本

版本使用语义化版本。`core.min` 写明确版本，例如 `1.2.3`。修复 bug 提升 PATCH；新增设置或模板能力提升 MINOR；删除字段或改变行为语义提升 MAJOR。

## 11. 安全要求

- 不修改 Core。
- 不读取服务器任意文件。
- 不暴露绝对路径、`.env`、配置或 Secret。
- 不绕过付费内容、发卡、评论权限。
- 不把媒体库原文件当作主题卸载对象删除。
- 外部资源 URL 必须可控并可降级。

## 12. 最小主题 Demo

```php
<?php

declare(strict_types=1);

use Cms\Core\Theme\TemplateContext;

/** @var TemplateContext $context */
?>
<!doctype html>
<html lang="zh-CN">
<head>
    <meta charset="utf-8">
    <title><?= $context->e($context->get('title', '')) ?></title>
</head>
<body>
    <main>
        <h1><?= $context->e($context->get('title', '')) ?></h1>
        <?= $context->get('rendered_blocks', '') ?>
    </main>
</body>
</html>
```

## 13. 发布前检查

- `theme_id` 与目录名一致。
- `theme.json` 能被 JSON 解析。
- 首页、列表页、内容页、404 都能渲染。
- 所有文本输出已转义。
- 图片设置不会阻塞保存其他设置。
- ZIP 只有单独根目录。

## 14. 提交审核流程

1. 开发主题。
2. 本地测试。
3. 生成符合 Market 标准的主题包。
4. 提交 Daiying 官方审核。
5. AI / 自动安全检查。
6. 官方人工审核。
7. 审核通过后发布。
8. 商业主题可按政策购买商业授权码。
