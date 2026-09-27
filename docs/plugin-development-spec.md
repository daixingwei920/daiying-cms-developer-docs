# Legacy / Superseded Plugin Development Spec

Last updated: 2026-09-27

Status: Legacy / Superseded.

This page previously contained the 2026-08-31 Daiying CMS V1.2 plugin development notes. It is no longer the current plugin development guide.

Current source of truth:

- [Daiying Plugin SDK V1 — Core 1.2.70+](https://github.com/daixingwei920/daiying-cms/blob/main/docs/plugin-sdk/README.md)

Current Plugin SDK V1 documentation set:

- [Development Specification](https://github.com/daixingwei920/daiying-cms/blob/main/DAIYING_PLUGIN_DEVELOPMENT_SPEC_V1.md)
- [API Reference](https://github.com/daixingwei920/daiying-cms/blob/main/DAIYING_PLUGIN_API_REFERENCE_V1.md)
- [Event Registry](https://github.com/daixingwei920/daiying-cms/blob/main/DAIYING_EVENT_REGISTRY_V1.md)
- [Capability Registry](https://github.com/daixingwei920/daiying-cms/blob/main/DAIYING_CAPABILITY_REGISTRY_V1.md)
- [Official Plugin Skeleton](https://github.com/daixingwei920/daiying-cms/tree/main/DAIYING_OFFICIAL_PLUGIN_SKELETON_V1)

Use Core 1.2.70 SDK APIs such as `rawBody()`, `content()`, `frontUsers()`, `license()`, `registerBlockRenderer()`, and `registerScheduledTask()`.

Do not use this legacy page to infer current APIs, event names, capabilities, or private Core access rules.
