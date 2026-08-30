# Android 构建与 Google Play

Play 比 App Review 宽容，但它卡发布往往是因为表单资料而不是代码——而这些卡点很容易撞上。

## 一次性设置

**第一次在 Play Console 创建应用只能手动。** 没有 API 途径。

两个不可撤销的选择：
- **"免费"上架后无法改成付费。**
- 包名和 iOS 的 bundle ID 一样是永久的。

## 签名，以及那个坑到所有人的 SHA-1

你用**上传密钥**签名；Play 会用它自己的**应用签名密钥**重新签名，而那把密钥要等你第一次上传 AAB 之后才生成。

```bash
keytool -genkey -v -keystore ~/app-release-key.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias app
```

keystore 不要放进仓库。然后：

**如果你用 Google 登录，Android OAuth 客户端上需要两个 SHA-1 指纹**——你的上传密钥*和* Play 的应用签名密钥（Play Console → 应用签名）。漏掉第二个，Google 登录就只在 Play 版本里坏掉，而你本地构建一切正常。它也没法在模拟器上测，因为 debug 的 SHA-1 同样没注册。功能测试必须用 Play 签名的构建装在真机上。

## 编译

```bash
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools   # 或你的 SDK 路径
eas build --platform android --profile production --local --output ./app.aab
```

没有 `ANDROID_HOME`，Gradle 会报 "SDK location not found"。

**没有密钥地图会崩。** `react-native-maps` 在 Android 上用 Google Maps，缺了 `app.json > android.config.googleMaps.apiKey` 就会**在原生初始化时崩溃**。iOS 不受影响，因为那边默认是 Apple Maps——这恰恰是它能悄无声息上线的原因。确认密钥进去了：解压 AAB，在 manifest 里找 `com.google.android.geo.API_KEY`。

## 上传

拖拽上传能用，但会永远停留在手动；一个典型 AAB 超过 60 MB，远超任何浏览器自动化的上限。用 Play 服务账号加 `eas.json > submit.android` 自动化。

### 你会遇到的发布错误

- **"此版本无法向现有用户提供，因为它不允许他们升级到新增的应用软件包。"** → 提高 version code，或者更好的做法是先走内部测试/封闭测试发布。
- **"此版本未添加或移除任何应用软件包。"** → AAB 没有正常上传。检查 version code 并重新上传。
- **原生调试符号** 必须是一个 `native-debug-symbols.zip`，里面按 ABI 分目录——`armeabi-v7a/`、`arm64-v8a/`、`x86_64/`，每个里面放 `libapp.so`——而且**不能有 `__MACOSX` 或 `.DS_Store` 条目**。

## 会卡住发布的声明

**广告 ID。** 解压 AAB 找 `com.google.android.gms.permission.AD_ID`。Firebase Analytics 需要这个权限并且声明要填"使用"；没有广告的应用两者都不该有。**规则是声明必须和 manifest 完全一致**——任一方向不一致都会卡住发布，而 Play 自己的提示文案有时会让你搞错是哪一边错了。

**隐私政策 URL。** 必须返回 200。我们第一次生产提交被拒，唯一原因就是声明的 URL 返回 404；应用本身没有任何问题。

**数据安全表单和内容分级问卷。** 上生产之前两者都是必填。按应用真实行为回答；它们会和你声明的权限相互核对。

**分发国家/地区。** 检查一下。我们有个应用在生产环境里被限制在**单一国家**，而 iOS 那边发行到 175 个国家——这不是任何人会故意选择的状态。

## 敏感权限

后台定位和 `FOREGROUND_SERVICE_LOCATION` 会触发一项 Play 权限声明，要求提供**演示视频**并接受审核。如果你现在还不需要，就明确屏蔽掉，而不是发上去然后卡住：

```json
"android": { "blockedPermissions": ["android.permission.ACCESS_BACKGROUND_LOCATION",
                                    "android.permission.FOREGROUND_SERVICE_LOCATION"] }
```

以后再有意识地加回来，同时准备好声明和视频。

## 目标 API 级别的截止日期

对于错过目标 API 级别提升截止日期的应用，Play 会停止接受更新。这个日期每年都会往后挪。**盯着它**——在发版当天才发现是很糟糕的一天。

## 关于 Play 速度的一点提醒

Play 审核很快，而这把刀是双刃的：一个有问题的版本可能一小时左右就上线，而且**撤不回来**。我们有个版本带着登录页崩溃上线了，唯一的补救就是推一个修好的 version code 然后等。先用内部测试。发布之后盯 Play Vitals 的崩溃数——我们就是这样确认修复生效的（10 次崩溃 → 0）。
