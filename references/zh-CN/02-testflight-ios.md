# iOS 构建与 TestFlight

我们每次构建都要走一遍的循环。大约 15 到 40 分钟；下面这些坑一旦漏掉，代价通常是一整轮。

## 编译之前先查这两件事

两件都很便宜，事后才发现就很贵。

1. **这个构建号是不是已经被占了？** `GET /v1/builds?filter[app]={id}` —— 重复的构建号在上传时会被拒。
2. **当前版本字符串是不是已经上架了？** 如果 App Store 上该版本处于 `READY_FOR_SALE`，它的发布列车就已关闭，上传会以 **90186** / **ITMS-90062** 失败。你必须提升**版本字符串**（不只是构建号）并**重新编译**——版本号是编译进 IPA 里的。

我们在某次发布上为此浪费了四个构建。

## 版本号实际存在哪里

- **managed workflow：** `app.json` → `expo.version` 和 `expo.ios.buildNumber`。
- **bare workflow：** `ios/<App>/Info.plist` → `CFBundleShortVersionString` 和 `CFBundleVersion`，外加 pbxproj 的 `MARKETING_VERSION`。**`app.json` 会被忽略。**

如果你用脚本改 `app.json`，要解析后再序列化（`json.load` / `json.dump`）。用同一个文件句柄读完再写会把文件截断——这让我们损失了一个构建。

## 编译

```bash
npx tsc --noEmit
npx expo export --platform ios --output-dir /tmp/exportcheck   # 在 archive 之前抓出 import 错误

LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
  eas build --platform ios --profile production --local \
  --non-interactive --output ./app-buildNN.ipa
```

`LANG` 前缀不是可选的。在 Ruby 4.0 加 CocoaPods 1.16 的组合下，`pod install` 会以 `Unicode Normalization not appropriate for ASCII-8BIT` 挂掉，项目里有非 ASCII 字符时尤其如此。

直接用 `xcodebuild` 也可以，而且当 EAS 凭证过期时这就是逃生通道：

```bash
LANG=en_US.UTF-8 xcodebuild -workspace App.xcworkspace -scheme App \
  -configuration Release -archivePath /tmp/AppNN.xcarchive \
  -allowProvisioningUpdates \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8 \
  -authenticationKeyID <KEY_ID> -authenticationKeyIssuerID <ISSUER_ID> archive
```

找 `ARCHIVE SUCCEEDED`，然后用 `method=app-store`、`signingStyle=automatic` 的 `ExportOptions.plist` 执行 `-exportArchive`。

### 编译进行中

**archive 期间不要改源文件。** Metro 会把一个写到一半的 bundle 打进去，应用启动就崩。这不是理论——它让我们损失了一个构建，而且看起来像个莫名其妙的崩溃。

## 上传

```bash
xcrun altool --upload-app -f ./app-buildNN.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
```

把 `.p8` 放在 `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`，altool 会从那里找到它。大约 15 秒完成。

**优先用它而不是 `eas submit`。** 我们见过 `eas submit` 卡 23 分钟没有任何输出也没有上传，也见过它以 "Unable to upload archive. Failed to authenticate for session" 失败。altool 会报出真正的错误。

## 上传之后

**`UPLOAD SUCCEEDED` 不代表被接受。** 轮询 ASC API 直到构建变成 `VALID`；Apple 在处理阶段也会拒。一旦有效，就把上一个构建设为过期，让测试者只看到新的：

```
PATCH /v1/builds/{id}   {"expired": true}
```

## 值得认识的上传错误

| 代码 | 含义 |
|---|---|
| **90062** / ITMS-90062 | 该版本已上架 —— 提升版本字符串 |
| **90186** | 预发布列车已关闭 —— 同一个原因 |
| **90713** | 某个 target 缺 `CFBundleIconName` —— Watch 和小组件需要自己的图标 |
| **ITMS-90863** | Apple Silicon 符号警告。**Expo 应用里很正常，不是拒绝。** 忽略它。 |

## 额外的 target

Watch 和 Live Activity 小组件需要在 `credentials.json` 里各自的描述文件，全部挂在同一张分发证书下。归档 Watch target 需要 Mac 上装有 watchOS **真机**平台，模拟器 SDK 不够：

```bash
xcodebuild -downloadPlatform watchOS    # 约 4 GB
```

**要测试额外 target 自己的流程。** 我们 Watch target 的登录从来就没通过——它发送 `email` 而后端读的是 `identifier`——Apple 在一次 2.1 拒绝里比我们先发现了。

## Xcode 在项目中途升级时

自动升级会让构建以 `iOS <版本> Platform Not Installed` 失败，哪怕当天早上还好好的：

```bash
xcodebuild -downloadPlatform iOS   # 约 8.5 GB，不需要 sudo
xcodebuild -runFirstLaunch
```

## 清理

EAS 的临时目录会无限增长——我们那里 `/var/folders/.../eas-cli-nodejs` 涨到了 35 GB。磁盘满了构建会以 `No space left` 失败。每次发布之间清一下。失败的尝试导致构建号跳号是正常的。

下一步：`04-app-store-submission.md`。
