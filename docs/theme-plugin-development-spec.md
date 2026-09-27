# Legacy / Superseded Theme And Plugin Development Spec

Last updated: 2026-09-27

Status: Legacy / Superseded.

This historical combined spec is no longer the current developer entrypoint.

Current developer documentation:

- [Daiying Plugin SDK V1 — Core 1.2.70+](https://github.com/daixingwei920/daiying-cms/blob/main/docs/plugin-sdk/README.md)
- [Theme Development Specification V1 Proposal — Core 1.2.70+](https://github.com/daixingwei920/daiying-cms/blob/main/DAIYING_THEME_DEVELOPMENT_SPEC_V1_PROPOSAL.md)
- [Theme API V1](https://github.com/daixingwei920/daiying-cms/blob/main/THEME_API_V1.md)
- [Daiying Theme Framework](https://github.com/daixingwei920/daiying-cms/tree/main/theme-framework)

The older 2026-08-31 content below is retained only for historical context.

---

# Daiying CMS V1.2 主题与插件开发规范

版本：2026-08-31  
适用版本：Daiying CMS V1.2  
适用对象：第三方主题作者、插件作者、官方扩展开发者  
状态：历史记录，已被上方当前开发者文档入口取代

## 1. 基本原则

Daiying CMS 的扩展分为主题和插件。

主题负责前台页面渲染、样式、模板结构、响应式体验和主题设置读取。主题不拥有业务数据，不实现支付、库存、订单、授权、发卡、云存储等功能规则。

插件负责注册区块、路由、后台菜单、事件、任务、Provider、配置、密钥和插件自己的数据。插件可以向主题提供渲染好的 HTML 或稳定 ViewModel，但不要求主题依赖插件私有数据库结构。

扩展不得修改 `system/core`、`config`、`storage`、`public/index.php` 等核心文件。主题和插件必须放在自己的独立目录中，并通过 manifest 声明身份、版本和兼容性。

## 2. 主题开发摘要

主题放在：

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

`theme_id` 必须与目录名一致。规则：小写字母开头，只允许小写字母、数字、下划线，长度 3 到 64。禁止使用 `safe`。

`theme.json` 示例：

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

模板中会得到 `$context`：

```php
use Cms\Core\Theme\TemplateContext;

/** @var TemplateContext $context */
```

常用方法：

```php
$context->get('title', '');
$context->setting('primary_color', '#1f6feb');
$context->e($value);
```

必须使用 `$context->e()` 输出不可信文本。Core 已渲染和清洗过的 `rendered_blocks` 可以作为 HTML 输出：

```php
<article><?= $context->get('rendered_blocks', '') ?></article>
```

主题资源推荐通过主题资源 URL 加载：

```text
/extension-assets/theme/{theme_id}?file=assets%2Flogo.png&v={version}
```

图片设置如果使用 `type: "image"`，主题应兼容字符串 URL、媒体对象、`uploads/...` 相对路径，并白名单过滤 URL。官方主题如已自带 Logo，不应强迫用户填写 Logo URL。

完整主题规范见首页 `#developers-theme` 区域，Markdown 文件为 `docs/theme-development-spec.md`。

## 3. 插件开发摘要

插件放在：

```text
content/plugins/{plugin_id}/
```

最小结构：

```text
content/plugins/vendor.demo/
  plugin.json
  plugin.php
  src/
  migrations/
```

`plugin_id` 必须与目录名一致。规则：小写字母开头，允许小写字母、数字、下划线、短横线、点号，推荐 `vendor.demo` 命名空间格式，长度不超过 96。第三方插件不得使用 `official.` 前缀，不得使用 `core`、`cms`、`cmsd`、`admin`、`system`、`market` 等保留命名空间冒充核心。

`plugin.json` 示例：

```json
{
  "plugin_id": "vendor.demo",
  "name": "Vendor Demo",
  "version": "1.0.0",
  "author": "Your Name",
  "core": {"min": "1.2.3", "max": "1.x"},
  "php": ">=8.1",
  "entry": "plugin.php",
  "trust_level": "api",
  "capabilities": ["blocks.register", "vendor.demo.manage"],
  "dependencies": []
}
```

`entry` 指向的 PHP 文件必须返回 callable：

```php
<?php

declare(strict_types=1);

use Cms\Core\Plugin\PluginContext;

return static function (PluginContext $context): void {
    $context->registerBlock('demo', 'Demo Block');
};
```

常用 `PluginContext` 能力：

```php
$context->hasCapability('blocks.register');
$context->listen('event.name', $handler);
$context->registerBlock('demo', 'Demo Block');
$context->frontRoute('GET', '/demo', $handler);
$context->adminRoute('POST', '/admin/demo/save', $handler, 'vendor.demo.manage', true);
$context->adminMenu('Demo', '/admin/demo', 'vendor.demo.manage');
$context->data()->put('setting', 'main', ['enabled' => true]);
$context->secrets()->set('vendor.demo', 'api_key', $apiKey);
```

完整插件规范见首页 `#developers-plugin` 区域，Markdown 文件为 `docs/plugin-development-spec.md`。

## 4. Capability

Core 已知能力：

```text
content.read
content.write
media.read
media.write
network.external
cron.register
settings.read
settings.write
storage.plugin
blocks.register
```

插件自定义能力必须落在自己的命名空间下，例如 `vendor.demo.manage`。不要声明不需要的能力；能力越多，审核范围越大。

## 5. 路由与区块

插件可注册前台和后台路由。后台写操作必须使用 POST 并启用 CSRF。前台写操作、支付回调、Webhook 必须验签、限流并具备幂等处理。

禁止覆盖：

```text
/
/{slug}
/admin/login
/recovery
/diagnostics
/health
/install
/admin/update
/api/market*
/admin/market*
```

插件区块应保存 JSON 可编码的数据，并带有 `plugin_id` 表明归属。插件停用或卸载后，内容中的区块数据必须保留，Core 显示缺失扩展占位，不得破坏文章内容。

## 6. 数据、迁移与 Secret

第三方插件默认使用 `PluginDataStore` 保存数据。需要独立表时，应在 `migrations/` 提供迁移，表名使用插件自己的前缀，并兼容 SQLite/MySQL。

API Key、Token、Secret 只能写入 secret store：

```php
$context->secrets()->set('vendor.demo', 'api_key', $apiKey);
$plain = $context->secrets()->get('vendor.demo', 'api_key');
$masked = $context->secrets()->masked('vendor.demo', 'api_key');
```

密钥不得写入 `plugin.json`、源码、日志、异常、队列 payload 或导出文件。

## 7. ZIP 与 Market 标准包

本地主题 ZIP 必须包含一个单独根目录：

```text
mytheme/
  theme.json
  templates/home.php
```

本地插件 ZIP 必须包含一个单独根目录：

```text
vendor.demo/
  plugin.json
  plugin.php
```

安装限制：

- ZIP 最大约 10 MB。
- 主题解压后最大约 25 MB。
- 插件解压后最大约 50 MB。
- 文件数不超过 500。
- 目录深度不超过 8。
- 不允许路径穿越、绝对路径、Windows 盘符路径。
- 不允许软链接、特殊文件、嵌套 ZIP。
- 不允许高风险可执行文件。
- 不允许写入 Core、配置、上传、其他主题或其他插件目录。

Market 标准包必须声明扩展 ID、类型、版本、兼容性、依赖和文件 SHA-256。文件路径必须位于对应扩展目录下。

当前正式发布流程：

1. 开发扩展。
2. 本地测试。
3. 生成符合 Market 标准的扩展包。
4. 提交 Daiying 官方审核。
5. AI / 自动安全检查。
6. 官方人工审核。
7. 审核通过。
8. 发布。
9. 商业扩展可按政策购买商业授权码。

## 8. 商业授权码政策

欢迎开发者为 Daiying CMS 开发插件和主题。

第三方开发者开发的插件或主题通过 Daiying 官方审核并获准发布后，可以按照开发者优惠价格购买该扩展对应的商业授权码。

当前开发者优惠价格：人民币 `¥1 / 个授权码`。

开发者购买授权码后，可以自行销售自己的插件或主题，并自行决定自己的最终销售价格。Daiying 平台提供扩展审核、授权码生成和授权验证基础设施。

`¥1 / 个` 是当前开发者购买商业授权码的优惠政策，不是插件售价、不是主题售价，也不是 Daiying 抽成。未来如有调整，以 Daiying 官方最新公布政策为准。

## 9. 安全要求

所有扩展必须遵守：

- 输出用户内容必须转义。
- URL、颜色、CSS 值必须白名单校验。
- 后台写操作必须 POST + CSRF。
- 前台回调必须验签、限流、幂等。
- 不在前台暴露内部错误、路径、密钥、堆栈。
- 不读取或写入其他插件的数据。
- 不修改 Core 文件。
- 不绕过支付、发卡、内容权限。
- 不打包测试密钥、真实 API Key、数据库密码。
- 不用 GET 执行删除、支付确认、状态变更等危险操作。
- 不把媒体库原文件当作插件卸载对象删除。
- 外部网络请求必须有超时、错误处理和明确 User-Agent。
- 插件停用、缺失或外部服务不可用时，历史内容和历史订单必须可读。

## 10. 发布前自检

主题自检：

- `theme_id` 与目录名一致。
- `theme.json` 能被 JSON 解析。
- 至少包含 `templates/home.php`。
- 首页、列表页、内容页、404 都能渲染。
- 所有文本输出已转义。
- 图片设置不会阻塞保存其他主题设置。
- CSS/JS 不依赖后台资源。
- ZIP 只有单独根目录。

插件自检：

- `plugin_id` 与目录名一致。
- `plugin.json` 能被 JSON 解析。
- `plugin.php` 返回 callable。
- capabilities 最小化。
- 后台 POST 路由启用 CSRF。
- Webhook 已验签、限流、幂等。
- 不使用保留路由。
- 不写 Core 文件。
- 不在源码中包含密钥。
- 卸载后保留用户内容和业务数据。
- Market 包内文件路径和 SHA-256 正确。

## 11. 审核口径

审核优先确认：

- 是否修改或依赖 Core 私有实现。
- 是否正确声明 ID、版本、兼容性和能力。
- 是否存在 XSS、CSRF、SSRF、路径穿越、密钥泄露风险。
- 是否能在插件停用、缺少依赖、外部 API 故障时降级。
- 是否保留用户内容、媒体和历史业务数据。
- 是否符合 ZIP 和 Market 包路径规范。

不符合扩展边界的实现应退回重构，不能通过“功能需要”作为理由进入 Core 私有目录。
