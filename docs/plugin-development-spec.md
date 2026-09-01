# Daiying CMS V1.2 插件开发规范

版本：2026-08-31  
适用版本：Daiying CMS V1.2

## 1. 插件目录

插件安装目录：

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

`plugin_id` 必须与目录名一致。

## 2. plugin.json

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
  "dependencies": [],
  "required_plugins": [],
  "table_prefixes": ["demo_"],
  "media_reference_provider": ""
}
```

第三方插件默认 `trust_level` 为 `api`。`trusted_php` 只允许官方可信内置插件使用。

## 3. plugin_id

规则：

- 小写字母开头。
- 允许小写字母、数字、下划线、短横线、点号。
- 推荐命名空间格式，例如 `vendor.demo`。
- 长度不超过 96。
- 第三方插件不得使用 `official.` 前缀。
- 不得使用 `core`、`cms`、`cmsd`、`admin`、`system`、`market` 等保留命名空间。

## 4. entry

入口文件必须返回 callable：

```php
<?php

declare(strict_types=1);

use Cms\Core\Plugin\PluginContext;

return static function (PluginContext $context): void {
    $context->registerBlock('demo', 'Demo Block');
};
```

入口只做注册、绑定和轻量初始化。耗时任务、外部 API 请求、批量同步应放到任务、事件或后台操作中。

## 5. PluginContext

常用接口：

```php
$context->hasCapability('blocks.register');
$context->listen('event.name', $handler);
$context->registerBlock('demo', 'Demo Block');
$context->frontRoute('GET', '/demo', $handler);
$context->adminRoute('GET', '/admin/demo', $handler, 'vendor.demo.manage');
$context->adminRoute('POST', '/admin/demo/save', $handler, 'vendor.demo.manage', true);
$context->adminMenu('Demo', '/admin/demo', 'vendor.demo.manage');
$context->data()->put('setting', 'main', ['enabled' => true]);
$context->secrets()->set('vendor.demo', 'api_key', $apiKey);
```

`pdo()` 原始数据库访问只对官方可信插件开放，第三方插件不得依赖。

## 6. Capability

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

自定义能力必须在插件命名空间下，例如：

```json
["blocks.register", "vendor.demo.manage", "vendor.demo.export"]
```

只声明当前版本真实需要的能力。

## 7. Route

前台路由：

```php
$context->frontRoute('GET', '/demo/products', $handler);
```

后台路由：

```php
$context->adminRoute('POST', '/admin/demo/save', $handler, 'vendor.demo.manage', true);
```

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

后台写操作必须 POST + CSRF。Webhook 必须验签、限流、幂等。

## 8. Block

注册区块：

```php
$context->registerBlock('faq', 'FAQ');
```

推荐数据：

```json
{
  "type": "faq",
  "plugin_id": "vendor.demo",
  "data": {
    "items": [
      {"question": "问题", "answer": "答案"}
    ]
  }
}
```

区块数据必须可 JSON 编码。插件停用或卸载后，内容中的区块数据必须保留。

## 9. PluginDataStore

默认使用：

```php
$context->data()->put('setting', 'main', ['foo' => 'bar']);
$rows = $context->data()->all('setting');
```

插件不得读取或写入其他插件私有数据。卸载默认保留业务数据，永久清除必须单独 purge 确认。

## 10. Migration

需要独立表时：

- 在 `migrations/` 中提供迁移。
- 表名使用插件自己的前缀。
- 字段类型兼容 SQLite 和 MySQL。
- 不直接修改 Core 表结构。
- 迁移必须可重复检测，不因二次执行破坏数据。

## 11. Secret Store

```php
$context->secrets()->set('vendor.demo', 'api_key', $apiKey);
$plain = $context->secrets()->get('vendor.demo', 'api_key');
$masked = $context->secrets()->masked('vendor.demo', 'api_key');
```

密钥不得写入 `plugin.json`、源码、日志、异常、队列 payload 或导出文件。

## 12. ZIP

插件 ZIP 必须包含单独根目录：

```text
vendor.demo/
  plugin.json
  plugin.php
```

限制：

- ZIP 最大约 10 MB。
- 解压后最大约 50 MB。
- 文件数不超过 500。
- 目录深度不超过 8。
- 不允许路径穿越、绝对路径、软链接、特殊文件、嵌套 ZIP。
- 不允许高风险可执行文件。

## 13. Market Package

Market 标准包必须包含扩展 ID、类型、版本、兼容性、依赖和文件 SHA-256。文件路径必须位于对应插件目录下，不得安装到 Core、配置、上传、主题或其他插件目录。

## 14. 兼容版本

版本使用语义化版本。`core.min` 写明确版本，例如 `1.2.3`。新增能力、路由、表或 Provider 属于 MINOR；删除字段、删除路由、改变数据结构属于 MAJOR。

## 15. 安全要求

- 输出用户内容必须转义。
- 后台写操作必须 POST + CSRF。
- 前台回调必须验签、限流、幂等。
- 不暴露内部错误、路径、密钥、堆栈。
- 不修改 Core。
- 不绕过支付、发卡、内容权限。
- 不打包密钥。
- 外部网络请求必须有超时和错误处理。
- 插件停用或缺失时，历史内容和历史订单必须可读。

## 16. 最小插件 Demo

```php
<?php

declare(strict_types=1);

use Cms\Core\Http\Request;
use Cms\Core\Http\Response;
use Cms\Core\Plugin\PluginContext;

return static function (PluginContext $context): void {
    $context->registerBlock('demo', 'Demo Block');
    $context->adminMenu('Demo 设置', '/admin/vendor-demo', 'vendor.demo.manage');
    $context->adminRoute(
        'GET',
        '/admin/vendor-demo',
        static function (Request $request): Response {
            return Response::html('<h1>Demo 设置</h1>');
        },
        'vendor.demo.manage'
    );
};
```

## 17. 发布前检查

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

## 18. 提交审核流程

1. 开发插件。
2. 本地测试。
3. 生成符合 Market 标准的插件包。
4. 提交 Daiying 官方审核。
5. AI / 自动安全检查。
6. 官方人工审核。
7. 审核通过后发布。
8. 商业插件可按政策购买商业授权码。
