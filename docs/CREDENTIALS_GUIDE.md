# 凭证配置指南（含多应用）

`config/credentials.json`（本地私有，gitignore 排除）集中存放各平台凭证。
**多应用**时用 `apps` 键按包名映射凭证（优先于顶层字段）。

推荐将凭据集中放在仓库外（如 `~/Documents/biao-signing/android-stores/`），目录权限 `700`、私钥与配置文件权限 `600`，通过 `--credentials /绝对路径/credentials.json` 使用。只把实际取得凭据的商店加入配置；示例中的占位符不能当成已开通。无需在聊天或工单中粘贴密码、私钥。

## 各商店从哪里获取

| 商店 | 后台入口 | 需要提供到本机的内容 |
| --- | --- | --- |
| 华为 | 开发者管理 → ID、证书和 Profile → 凭证 → 开发者级凭证 → 服务账号凭证；AGC 旧界面为用户与权限 → API 密钥 → Connect API → Service Account | 下载的 `*private.json`，以及目标包名/应用 ID |
| 小米 | 管理中心 → 自动发布接口 | 登录邮箱、接口私钥、下载的公钥证书 |
| OPPO | 管理中心 → 产品 → 我的 API → 客户端应用 | 发布 API 的 `client_id/client_secret` |
| vivo（国内） | 账号 → 账号管理 → API 管理 → 开通 | `access_key/access_secret` |
| 荣耀 | 管理中心 → 开放能力 → 凭证 → 申请凭证 | `client_id/client_secret` |
| 应用宝 | 账号管理 → API 发布接口 → 申请开通 | `UserID/access_secret` 及应用 `app_id` |

使用具备相应权限的开发者主账号开通。这里需要的是**应用发布 API 凭据**，不是推送 SDK、手机号登录、广告或支付密钥。后台菜单可能随平台改版变化；[华为官方鉴权说明](https://developer.huawei.com/consumer/cn/doc/appgallery-connect-guides/agcapi-getstarted-0000001111845114/)、[小米官方自动发布指南](https://dev.mi.com/xiaomihyperos/documentation/detail?pId=1134)、[荣耀官方 API 指南](https://developer.honor.com/cn/doc/guides/101359)、[DCloud 多商店接入方说明](https://uniapp.dcloud.net.cn/uni-publish/config.html)提供对应入口。

## 通用结构

```json
{
  "google":   { "service_account_json": "/path/to/service-account.json" },
  "apple":    { "country": "cn", "apps": { "com.android.pkg": "1234567890" } },
  "xiaomi":   { "apps": { "com.android.pkg": { "email": "", "password": "", "public_key": "" } } },
  "oppo":     { "apps": { "com.android.pkg": { "client_id": "", "client_secret": "" } } },
  "vivo":     { "apps": { "com.android.pkg": { "access_key": "", "access_secret": "" } } },
  "honor":    { "apps": { "com.android.pkg": { "client_id": "", "client_secret": "" } } },
  "huawei":   { "apps": { "com.android.pkg": { "service_account_file": "/absolute/path/huawei-service-account.json", "app_id": "" } } },
  "qq":       { "user_id": "", "access_secret": "", "apps": { "com.android.pkg": "应用宝app_id" } }
}
```

## 各平台详细说明

### Google Play

| 字段 | 说明 |
| --- | --- |
| `google.service_account_json` | Google Cloud Console 服务账号 JSON 文件路径 |

获取：Google Cloud → 创建服务账号并下载 JSON → Play Console → 用户和权限 → 邀请服务账号（授予发布权限）。

### Apple App Store（查询 + 发布）

| 字段 | 说明 |
| --- | --- |
| `apple.country` | 地区（默认 cn） |
| `apple.apps` | 安卓包名 → 数字 App ID（发布必须）或 bundleId（仅查询）；也可写成 `{"app_id": 数字ID, "api_key": {...}}` |
| `apple.api_key` | 可选，**默认密钥**；只有所有应用都属于同一个开发者账号时才配置 |
| `apple.apps.<包名>.api_key` | 该应用专属的 App Store Connect 密钥（issuer_id / key_id / private_key_path） |

**不同开发者账号的 Key 不能混用**：一个 `.p8` 只能访问它所属账号下的应用。
应用分散在多个账号时，请逐个应用配置 `apps.<包名>.api_key`，不要配顶层 `api_key`；
顶层 `api_key` 只在该应用没有单独配置时作为回退。

App Store 链接里的数字 ID：`https://apps.apple.com/cn/app/id1234567890` → `1234567890`。
只查询时用 bundleId（如 `com.example.app`）也可以；**发布必须配数字 ID 和该账号的 api_key**。

```json
"apple": {
  "country": "cn",
  "apps": {
    "com.kaidishi.lock": 1155786023,
    "com.motorola.security": {
      "app_id": 1234567890,
      "api_key": {
        "issuer_id": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
        "key_id": "NPLV7ZY2R5",
        "private_key_path": "/Users/you/keys/app_stores/AuthKey_NPLV7ZY2R5.p8"
      }
    }
  }
}
```

- `.p8` 只在创建 Key 时可下载一次，丢失需吊销重建
- Key 的角色需要 **App Manager**（或 Admin/Account Holder），否则拉构建会 403
- 发布走「选择 TestFlight 已上传构建 → 关联版本 → 提交审核」，本机不需要 ipa
- `apps` 里没有数字 ID 或没有可用密钥的应用，页面不会出现「苹果 App Store 信息」区块

### 小米应用商店

```json
"xiaomi": {
  "apps": {
    "com.example.app": {
      "email": "账号邮箱",
      "password": "自动发布接口分配的私钥",
      "public_key": "/绝对路径/dev.api.public.cer"
    }
  }
}
```

- `public_key`：小米开放平台下载的 X509 公钥证书（.cer/.pem），用于 RSA 加密签名
- `password` 是历史配置字段名，填写小米**接口私钥**，不是小米登录密码
- 不同账号对应不同证书/私钥，按包名分别配置

### OPPO 软件商店

```json
"oppo": { "apps": { "com.android.pkg": { "client_id": "19位", "client_secret": "..." } } }
```

- client_id 为 **19 位数字**（OPPO 开放平台 → 产品 → 我的 API；不是应用推送密钥）
- 更新版本自动复用现有资料，无需重复传图标/截图

### vivo 应用商店

```json
"vivo": { "apps": { "com.android.pkg": { "access_key": "", "access_secret": "" } } }
```

- 开放平台申请 API 传包服务后获得 access_key/access_secret

### 荣耀应用市场

```json
"honor": { "apps": { "com.android.pkg": { "client_id": "", "client_secret": "" } } }
```

- 荣耀开发者服务平台 → 凭证（client_id / 密钥）

### 华为 AppGallery：Service Account（推荐）

1. 由账号持有人在上表入口创建**开发者级**服务账号，授予目标应用上传、发布/查询所需角色（例如 APP 管理员，以后台当前角色权限为准）。无需提供账号密码给 CLI。
2. 点击生成公私钥并下载 JSON。文件应包含 `key_id`、`private_key`、`sub_account`；若没有生成密钥，下载文件可能缺少私钥，需要在后台补齐。
3. 将下载文件保存在本机仓库外，例如 `~/Documents/biao-signing/android-stores/huawei-service-account.json`，无需编辑 JSON 内容。
4. 配置下方 `service_account_file`，指向该文件。支持绝对路径和 `~`；相对路径以命令执行目录为准，建议使用绝对路径以避免切换目录后找不到文件。

```json
"huawei": {
  "apps": {
    "com.android.pkg": {
      "service_account_file": "~/Documents/biao-signing/android-stores/huawei-service-account.json",
      "app_id": "应用数字ID"
    }
  }
}
```

- CLI 使用现有 `cryptography` 在本机签署 PS256 JWT，直接作为 Bearer；无需额外配置 `client_id/client_secret`，不调用旧 token 交换接口。
- JWT 有效期一小时，缓存按包名隔离，到期前刷新，收到 HTTP 401 时强制刷新后最多重试一次。
- 同时出现新旧认证字段时使用 `service_account_file`；文件不存在、格式或私钥错误会直接失败，不静默换用旧账号。
- `apps` 存在时必须有明确包名条目；凭证对象不自动继承其它应用或顶层认证。兼容例外是旧 `apps[包名] = "client_id"` 字符串简写，它仍使用顶层 `client_secret`。多个应用可显式引用同一开发者级服务账号文件，但该账号必须有各应用相应权限。
- 单应用也可在 `huawei` 顶层直接写 `service_account_file`；Android 根据 APK 包名查询 appId，Harmony 仍需显式 `app_id`。
- 修改/轮换凭据后重启长期运行的 Web 服务，让新凭据立即生效。

### 华为旧 API 客户端兼容

已有发布 API 客户端仍可用原配置：

```json
"huawei": {
  "apps": {
    "com.android.pkg": {
      "client_id": "已有发布API客户端ID",
      "client_secret": "已有发布API客户端密钥",
      "app_id": "应用数字ID"
    }
  }
}
```

旧方式继续调用 `/api/oauth2/v1/token` 并携带 `client_id` 请求头。不要使用普通应用的 OAuth 登录/推送凭据。华为官方建议新建凭据选 Service Account，并迁移旧客户端；权限取决于实际凭据级别和角色，不再把所有凭据都当作 `project_client_id`。

### 华为 Android / Harmony 多应用示例

```json
"huawei": {
  "apps": {
    "com.example.android": { "service_account_file": "/secure/huawei-service-account.json", "app_id": "100000001" },
    "com.example.harmony": { "service_account_file": "/secure/huawei-service-account.json", "app_id": "100000002", "app_kind": "harmony" }
  }
}
```

Harmony 包名须单独配置并标记 `app_kind: "harmony"`。如需现有 Android/Harmony 联合发布入口，同时在应用目录配置 `huawei_harmony_package`。本次仅增加认证支持，Harmony 仍选择 AGC 已上传构建，不从本机上传新 `.app/.hap`。

### 腾讯应用宝

```json
"qq": {
  "user_id": "开发者UserID",
  "access_secret": "API发布接口密钥",
  "apps": { "com.example.app": "应用宝app_id" }
}
```

CLI 平台名是 `qq`。当前适配器只支持已上线应用的更新；首次创建/发布在应用宝后台完成。

## 校验凭证

```bash
appstore --json validate --credentials /absolute/path/credentials.json
appstore --json status --platform huawei --package com.example.app --credentials /absolute/path/credentials.json
```

`validate` 是本地配置检查；华为会校验服务账号文件、必要字段和 RSA 私钥，但不验证后台权限。`status` 才会请求商店的只读查询接口，可用于确认账号权限。不要以本地检查通过代替真实鉴权成功。

华为 `publish --dry-run` 仅本地校验（含 Android/Harmony），不联网、不提审。其它商店 dry-run 行为不同：Google 会创建并删除 edit，应用宝会调用只读接口，详见[架构说明](ARCHITECTURE.md#dry-run-行为)。实际 `publish` 会上传并提交审核，应在明确发布范围后使用。
