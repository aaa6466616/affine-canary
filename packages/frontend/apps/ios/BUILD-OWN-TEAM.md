# 用自己的 Apple 团队构建 iOS 包

这个 fork 把 iOS 工程的签名改成了 `com.feihu1201.iaffine` / 团队 `8DQ2M68TNF`，
并用独立的 GitHub Actions workflow（`.github/workflows/ios-build.yml`）在
macOS runner 上构建、签名、导出 IPA，可选上传 TestFlight。

上游 `release-mobile.yml` 里的 iOS job 依赖 TOEVERYTHING 的证书和
`app.affine.pro` 的 App ID，无法复用，不要手动触发它。

## 改动清单

| 位置 | 上游值 | 现在 |
| --- | --- | --- |
| App bundle id | `app.affine.pro` | `com.feihu1201.iaffine` |
| ShareExtension | `app.affine.pro.ShareExtension` | `com.feihu1201.iaffine.ShareExtension` |
| 单元测试 | `app.affine.pro.tests` | `com.feihu1201.iaffine.tests` |
| Development Team | `73YMMDVT2M` | `8DQ2M68TNF` |
| App Group | `group.app.affine.pro` | `group.com.feihu1201.iaffine` |
| 签名方式 | Manual（写死 TOEVERYTHING 证书） | Automatic |

App Group 同时改了 `App/App/App.entitlements`、`ShareExtension.entitlements`
和 `App/Shared/ShareInbox/ShareInboxConstants.swift` 三处，必须保持一致，
否则分享扩展和主 App 的收件箱对不上。

## 需要的 GitHub Secrets

| Secret | 用途 |
| --- | --- |
| `APPLE_STORE_CONNECT_API_KEY_ID` | App Store Connect API Key 的 Key ID |
| `APPLE_STORE_CONNECT_API_ISSUER_ID` | API Key 的 Issuer ID |
| `APPLE_STORE_CONNECT_API_KEY` | `.p8` 私钥文件的 **base64** |
| `BUILD_CERTIFICATE_BASE64`（可选） | Apple Distribution 证书 `.p12` 的 base64 |
| `BUILD_CERTIFICATE_PASSWORD`（可选） | 上面 `.p12` 的密码 |

不提供 p12 时，workflow 依赖 `xcodebuild -allowProvisioningUpdates`
自动下载/生成描述文件；如果 runner 报找不到签名证书，就补上这两个 secret。

### 创建 App Store Connect API Key

App Store Connect → 用户和访问 → 集成 → App Store Connect API →
生成密钥（角色至少 App Manager）→ 下载 `.p8`（只能下载一次）→ 记录 Key ID 和 Issuer ID。

```bash
base64 -w0 AuthKey_XXXXXXXXXX.p8
```

### （可选）没有 Mac 也能做 Distribution 证书

Apple 门户 → Certificates → 新建 Apple Distribution，需要上传 CSR。
可以在 Windows 上用 OpenSSL 生成私钥和 CSR：

```bash
openssl req -new -newkey rsa:2048 -nodes -keyout dist.key -out dist.csr \
  -subj "/CN=Affine Distribution/O=feihu/C=CN"
```

把 `dist.csr` 上传到开发者门户，下载 `.cer`，然后合成 p12：

```bash
openssl x509 -inform DER -in distribution.cer -out distribution.pem
openssl pkcs12 -export -inkey dist.key -in distribution.pem -out dist.p12 -passout pass:你的密码
base64 -w0 dist.p12
```

## App ID / App Group

自动签名通常会在首次构建时自动创建 App ID 和 `group.com.feihu1201.iaffine`。
如果 CI 报 capability 错误，在开发者门户手动补：

- App ID：`com.feihu1201.iaffine`（Capabilities 勾选 App Groups）
- App ID：`com.feihu1201.iaffine.ShareExtension`（同样勾选 App Groups）
- App Group：`group.com.feihu1201.iaffine`，并把上面两个 App ID 都加进去

## 首次 TestFlight

TestFlight 上传要求 App Store Connect 里已经存在 bundle id 为
`com.feihu1201.iaffine` 的 App 记录。可以先在 App Store Connect → App 里
新建一个（名称随意，SKU 唯一），再跑 workflow。

首次构建上传后，TestFlight 会显示"缺少合规性信息"，在 App Store Connect
里回答一次（通常选"不包含加密算法"）即可开始测试。

## 触发构建

workflow 只支持手动触发（`workflow_dispatch`）：

- `build-type`：前端 bundle 类型，默认 `canary`
- `upload-to-testflight`：是否上传 TestFlight，默认是
- `marketing-version` / `build-number`：留空则使用工程里的版本号和 GitHub run number

构建产物：Actions 页面的 `affine-ios-ipa` artifact（保留 14 天）。

## 为什么不用无签名构建

iOS 不接受未签名 App，`CODE_SIGNING_ALLOWED=NO` 产出的 IPA 无法直接安装；
`App.entitlements` 里的 App Group 在免费个人团队下不可用，重签也会失败。
## 注意

- `packages/frontend/apps/ios/App/fastlane/Appfile` 已改为自己的
  app_identifier / team，`Fastfile` 的上传 lane 仍可用，但 workflow 用的是
  `xcodebuild` + `fastlane pilot upload`，不依赖上游的 `beta` lane。
- 上游 workflow 的每日定时任务已从 `release.yml` 和
  `copilot-test-automatically.yml` 移除，避免推送后自动跑一整套发布流程。
- 如果要对自建服务器联调 HTTP（非 HTTPS），iOS 侧还需要在 `Info.plist`
  配置 ATS 例外（`NSAppTransportSecurity`），目前工程没有放开。
