# 001 — 华为 Service Account 与发布凭据配置

日期：2026-11-22。用户授权 fork `junbin-lai/app-publish-hub` 并适配华为新接口，说明如何提供商店发布凭据。授权不包含实际商店提审。

## 目标与范围

1. 建立用户 fork 与独立 checkout，保留 upstream，变更提交使用 `[skip ci]`，不运行 GitHub Actions。
2. 在现有 `HuaweiAdapter` 增加开发者级 Service Account，复用已声明的 `cryptography`，兼容旧 `client_id/client_secret`。
3. 更新示例、架构和各商店凭据获取说明；配置仅引用本机凭据文件，不把私钥入库。
4. 编译检查、全部 diff 复核、独立安全复核后推送 fork；未获用户明确测试授权，不运行自动测试。

## 契约与性质清单

- 顶层 `huawei.service_account_file` 或 `huawei.apps.<包名>.service_account_file` 指向华为下载的 JSON，支持绝对路径和 `~`；相对路径以进程工作目录为准，文档推荐绝对路径。
- `apps` 存在时必须匹配明确包名，不回退到另一应用；保留旧字符串 client_id 简写。每条应用显式配置自己的认证方式。
- Service Account 文件包含 `key_id/private_key/sub_account`，私钥必须能解析为 RSA；读取、JSON、字段、密钥错误以 `StoreError` 返回，不回显文件内容或私钥。
- 使用 PS256（RSA-PSS/SHA-256，salt 长度 32 字节）生成 JWT；`kid=key_id`、`iss=sub_account`、`aud=https://oauth-login.cloud.huawei.com/oauth2/v3/token`、`exp=iat+3600`。JWT 直接作为 Bearer，不走旧 token 交换接口；新方式不要求或伪造 `client_id`。
- Service Account 配置存在时优先使用；失效时不静默切换旧账号。旧 API 客户端保持现有 token 请求和 client_id 请求头。
- 缓存按包名隔离、加锁、到期前刷新；HTTP 401 强制刷新后最多重试一次。Android v2 与现有 Harmony v3 共用鉴权。
- `check`、Android/Harmony dry-run 对新旧凭据做本地校验；不通过这次改动引入鉴权网络调用。

## 威胁模型

防止私钥误入 Git、异常输出泄露凭据、错配包名/账号、无效新凭据静默回退，以及鉴权失败被当作成功。沿用当前业务发布接口，不扩大到恶意本机环境或全仓安全改造。

## Non-goals

不自动创建服务账号或授予权限，不提交商店审核，不改应用功能，不扩充鸿蒙构建上传，不改变其它商店发布行为，不新增依赖，不触发 CI，不运行自动测试。

## 执行记录

- 基线：upstream main `9fee64ead5671c39905083495c5238fae18523b8`。
- 已核对华为[官方鉴权说明](https://developer.huawei.com/consumer/cn/doc/appgallery-connect-guides/agcapi-getstarted-0000001111845114/)，新方式为直接 Bearer JWT；原项目仅实现旧 token 交换。
- 主会话负责契约、fork、凭据文档、本机安装配置与最终验收。实现委派 `gpt-6.1-sol / medium`；独立安全复核使用 `gpt-6.1-sol / high`，理由分别为局部平台适配和认证/凭据隔离检查。文件写入范围互斥。
- 初始 GitHub 浏览器未登录；用户随后完成登录，已创建 [liuhuyu/app-publish-hub](https://github.com/liuhuyu/app-publish-hub) fork，保留 `upstream` 指向原项目。
- 已完成华为单文件适配、凭据指南与示例；原始历史接入记录保留，并加新旧认证适用范围声明。
- 独立复核 `gpt-6.1-sol / high` 未发现 P0/P1；P2“旧字符串简写继承顶层 secret”的文档例外已补齐。
- 本机独立 `.venv` 安装成功；私有应用目录和凭据引用已生成，真实密钥尚未提供。发布包路径留空，避免把旧构建当作待发布产物。
- 实现提交 `1d06bf61415249c1d92258d7fd1c8e1700c142f8` 已通过 SSH 推送 fork `main`；远端回读一致。未触发 GitHub Actions。

## 验证与未完成项

- `python3 -m py_compile app_store/stores/huawei.py` 通过；示例 JSON 解析与 `git diff --check` 通过。
- 独立虚拟环境 `appstore --help` 启动成功，`pip check` 未发现依赖冲突。
- 已逐项 review 全部 diff；私有配置、应用目录及华为下载私钥文件匹配 Git 忽略规则。
- 按用户规则未运行自动测试、未联网鉴权、未实际上传或提审。没有华为发布服务账号，不将本地验证记作真实 API 接入成功。
- 本次 fork、适配与凭据指引已完成。后续真实只读鉴权需要用户在华为后台创建并提供具备相应角色的开发者级服务账号 JSON；商店实际提审另行明确范围。
