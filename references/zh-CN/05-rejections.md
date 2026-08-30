# 拒绝案例库与提交前检查清单

这不是泛泛的商店建议。下面每一条都在上架八款应用的过程中真实发生过，并附上我们最终找到的根因——而根因往往不是拒绝通知上写的那个。应用做了匿名化：**应用 A**（社交活动）、**应用 B**（地点与地图指南）、**应用 C**（面向代理商的 B2B 工具）。

---

## 最贵的一课：App Review 备注里**不能**写什么

应用 A 的第 49 个构建被 **Guideline 4.2（Minimum Functionality）** 拒了。拒绝正文几乎逐字引用了我们自己写在审核备注里的句子：

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network. **Staying small is a core part of the product.**"

Apple 把这读成了"这东西属于 Ad Hoc 分发，不属于 App Store"。

**规则：永远不要这样描述你的应用**——"small"、"niche"、"few dozen"、"invite-only"、"not mass-market"、"private group"、"closed community"、"面向特定社群"。

**正确的表述：**"面向所有人、免费、任何城市或国家都能下载；精选会员层只是一个*可选*层"。就算产品真的是邀请制，它也需要一个**不用账号**就能看的面，而备注要描述的正是这一面。

另外：**审核备注上限 4000 个字符。** 超出会让 API 返回 409。

---

## 提交前检查清单

每一个方框都来自一次拒绝。逐条核对。

### 账号与数据（5.1.1）
- [ ] **账号删除。** 只要能创建账号，就必须能在应用内删除 —— 5.1.1(v)。*（在我们两款应用上都造成过拒绝。）* Apple 最终接受的流程满足了以下全部条件，每一条都重要：
  - **立即且永久。** 没有停用，没有冷静期。
  - **不能有客服/邮件/电话步骤，不能跳转到网页** —— Apple 把这些都算作"无法在应用内删除"。
  - 用密码重新认证，并给出破坏性操作确认。
  - 明确列出会删除什么；第三方授权（Instagram/Facebook）也要在对方平台侧撤销。
- [ ] **权限用途说明** —— 5.1.1(ii)。**bare workflow 陷阱：** 如果 `ios/` 目录在仓库里，`app.json` 里的说明**不会**同步到 `Info.plist`。手动检查。
- [ ] **不要申请你不用的权限。** 应用 B 被 5.1.1 拒了，因为 `pickImage` 在打开选择器之前申请了完整相册权限。现代 iOS 的 PHPicker **不需要任何权限**就能返回一张照片 → 删掉那次申请。
- [ ] **法律链接可点击** —— 2.1(a)。应用 C 栽在这里：注册页上"创建账号即表示你同意条款"是**纯文本**，登录页干脆没有链接，审核人员根本看不到条款。用**应用内浏览器**（`expo-web-browser`）打开，而不是把人甩去 Safari，并且登录页也要放。
- [ ] **定位权限前置引导页** —— 5.1.1(iv)。不要有倾向性文案："使用我的位置" ❌ → "继续" ✅。也不要两个让人困惑的"跳过"。

### 用户生成内容（1.2）—— 只要用户能发布任何东西就是强制的
应用 A 的第 51 个构建栽在这里。四项缺一不可：
- [ ] 一个**可见的**举报入口。长按手势**不够**：审核人员找不到。在每条消息、帖子和评论旁边放一个可见的"⋯"按钮。
- [ ] 拉黑（双向：被拉黑的人也不能给你写）。
- [ ] **所有**写入接口上的内容过滤，不只是那几个显眼的。
- [ ] 私信同意机制：发起方在对方接受之前只能发一条。

### 购买信号（3.1.1）—— 免费应用和 B2B 应用最阴的坑
应用 C 因为这一条被拒了**两次**。
- [ ] **不要留下任何价格、套餐名称、额度计数器、付费墙、升级按钮或站外购买链接。** 击沉第 25 个构建的，是一行"Intelligence 不在你当前套餐内——至少需要 Solo 套餐"、一个剩余额度计数器和一个"每个品牌 1 额度"的标签。光显示套餐名称就够了。
- [ ] **不要只靠 3.1.3(f)"Free Stand-alone Apps"这条论证——Apple 拒了。** 我们在第 26 个构建上试过。
- [ ] **一个公开的注册页会摧毁你的 B2B 主张。** 薄弱环节就在这里：一个"加入"页面在审核人员看来就是面向消费者的自助销售，和 3.1.3(c) 的 "only sold directly by you to organizations" 直接冲突。第 27 个构建的解法是**把注册页整个删掉**——应用只提供登录。

### 元数据
- [ ] **年龄分级** —— 2.3.6。任何带"结识他人 / 社交拓展"角度的应用，`matureOrSuggestiveThemes` 至少要是 `INFREQUENT_OR_MILD`。可以用 API 改：`PATCH /v1/ageRatingDeclarations/{id}`。
- [ ] **隐私政策 URL 返回 200 吗？** 我们第一次在 Play 上的生产提交被拒，唯一原因就是声明的 URL 返回 404。

### 审核人员实际会看到的东西
- [ ] **演示账号真的能用吗？** 在真机上试。应用 A 被 2.1 拒了，因为 Apple Watch target 的登录从来就没通过：它发送 `email` 而后端读的是 `identifier`，返回 422。没人发现，因为手机端 target 发的字段是对的。
- [ ] **演示数据新鲜吗？** 应用 A 里 16 个种子活动已经过期，审核人员打开会是一个空应用。准备一个幂等脚本把日期往前推。
- [ ] **验证墙会不会困住审核人员？** 如果注册但未验证的人什么都看不到，应用看起来就是封闭的。让访客能浏览，只在写入时要求验证。
- [ ] **不要上线空的或关掉的模块。** 应用 A 里一个关着的功能开关留下了一个空的"课程"板块，连续造成两次 2.1 拒绝（App Completeness，然后 Information Needed）。最后整个删掉了。**不要提交做了一半的功能——把它拿掉。**
- [ ] **你没法选择审核设备。** 我们收到过 iPad Air 和 Apple Watch 的反馈。主设备之外的 target 也要测。

### 平台集成（4.0 Design）
- [ ] **你的地图/定位功能和原生应用集成了吗？** 应用 B 被 4.0.0 拒了，因为它只是把用户甩给 Google Maps。要提供 Apple Maps（`maps.apple.com`）作为选项。

### Android
- [ ] **广告 ID 声明和 manifest 一致吗？** 解压 `.aab` 找 `com.google.android.gms.permission.AD_ID`。如果没有，声明就必须写"未使用"——声明填错会卡住发布。
- [ ] **分发国家/地区。** 我们有个应用在生产环境里意外只发到**一个国家**，而 iOS 发到 175 个。
- [ ] **后台定位**可能触发 Play 的定位权限声明。
- [ ] **Maps API 密钥** 放在 `app.json > android.config.googleMaps.apiKey`：没有它，`react-native-maps` 在 **Android 上会在原生初始化时崩溃**。iOS 不会（那边默认 Apple Maps），这正是它能溜过去的原因。
- [ ] **Google 登录需要两个 SHA-1：** 你的上传密钥**和** Play 的应用签名密钥。Play 要等第一次 AAB 上传之后才生成自己的密钥；如果那个 SHA-1 没加到 Android OAuth 客户端上，Google 登录就只在 Play 版本里坏掉。它也没法在模拟器上测（debug 的 SHA-1 同样没注册）：必须用 Play 签名的构建装在真机上。

---

## 拒绝案例库

| Guideline | 商店怎么说 | 真正的根因 | 解法 |
|---|---|---|---|
| **4.2** Minimum Functionality | "small, or niche, set of users" | **我们自己写在审核备注里的话** | 无需账号的发现流程 + 公开 API 端点 + 内容配比调整 |
| **1.2** UGC | 没有过滤 / 举报 / 拉黑 | 举报只藏在一个看不见的长按手势后面；私信和活动大厅里根本没有 | 8 个界面上可见的"⋯"菜单，9 个写入接口上的过滤，私信同意机制 |
| **2.1** 演示账号 | 无法登录 | Watch target 发 `email`，后端读 `identifier` → 422 | 修正字段；备注顶部加 "WATCH — PLEASE READ FIRST" |
| **2.1** App Completeness | "could not access the courses" | 功能开关关着，板块显示为空 | 功能整个移除 |
| **2.1** Information Needed | "你们的目标用户量是多少？" | 同一个空模块，加上 4.2 那套表述 | 备注重写 |
| **2.3.6** 年龄分级 | "Mature or Suggestive Themes" | 结识他人的主题没有申报 | 通过 ASC API `PATCH ageRatingDeclarations` |
| **3.1.1** 应用内购买 | 免费应用里出现购买信号 | 套餐名称、额度计数器、"需要 Solo 套餐"文案 | 清除所有价格与套餐痕迹 |
| **3.1.1**（第二次） | 同一条 guideline 又来一次 | 3.1.3(f) 论证不够；**公开的注册页**推翻了 B2B 主张 | 删掉注册页，只保留登录 |
| **2.1(a)** App Completeness | 看不到条款 | 法律文字是纯文本、不可点，登录页还没有 | 可点击链接，在应用内浏览器打开 |
| **5.1.1(v)** Data Collection | 没有账号删除 | — | 应用内账号删除 |
| **5.1.1(ii)** | 缺权限用途说明 | bare workflow 不会把 `app.json` 同步到 `Info.plist` | 直接改 `Info.plist` |
| **5.1.1(iv)** 定位流程 | 前置引导页有倾向性 | 按钮文案加上重复的跳过 | 改成"继续"，只留一个出口 |
| **5.1.1** 相册访问 | 申请了相册权限 | PHPicker 根本不需要那个权限 | 删掉权限申请 |
| **4.0.0** Design | "not integrated with built-in mapping" | 只甩给 Google Maps | 加上 Apple Maps 选项 |
| **Play**（生产） | 提交被拒 | 声明的隐私政策 URL 返回 404 | 加永久别名并修正控制台条目 |

**注意：** 修好一个拒绝可能招来下一个。有个应用连续被拒四次，另一个三次。每次修完，重新过一遍**整份**清单。

---

## App Store Connect API 与声明的坑

这些是用 API 填提交字段时遇到的：

- **不存在 `APP_IPHONE_69` 这个截图类型。** API 接受的最大 iPhone 类型是 `APP_IPHONE_67`（1290×2796）。为 6.9 英寸设备渲染的 1320×2868 图片会被**拒绝**：上传 6.7 英寸的，让 Apple 放大。
- **`whatsNew` 在首个版本上无法编辑** —— 409，"cannot be edited at this time"。它只存在于更新版本。
- **年龄分级的字段类型是混着的：** 有些是 BOOLEAN（`messagingAndChat`、`userGeneratedContent`、`advertising`），有些是 STRING 枚举（`contests`、`profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`）。类型填错返回 409，错误信息会告诉你正确的集合。
- **Apple 在 2025 年改了年龄档：12+ 已经没有了。** 现在是 4+、9+、13+、16+、18+。如实回答可能得到 4+；用 `ageRatingOverrideV2` 往上调（例如 `THIRTEEN_PLUS`）。

**App Privacy 声明：**
- **身份证号不属于 "Sensitive Info"。** Apple 的敏感类别涵盖种族、宗教、性取向、生物识别等；身份证号不在其中 → 正确的归类是 **"Other Data Types"**。
- **存在你自己数据库里的银行信息算 "Collected"。** 只有支付服务商持有、你无法访问时 Apple 才豁免。
- ⚠️ **闭眼点击陷阱：** 向导会按数据类型以不同高度渲染。重复点同一个位置产生过 "User ID 用于追踪：是" 这种错误答案。每一项都截图核对最终状态。

---

## 构建与上传的坑

### 版本列车
**你不能在一个已通过审核的版本字符串上传新构建** —— altool 报 90062 / 90186（"Invalid Pre-Release Train ... closed"）。提升 `app.json` 里的 `version` 并**重新编译**：版本字符串是打进 IPA 的。我们为此烧掉了一整个构建。

### 上传
- `eas submit` 可能卡住（超过 23 分钟，没有输出），或者以 "Failed to authenticate for session" 失败。**可靠的路子是直接用 altool：**
  ```bash
  xcrun altool --upload-app -f build.ipa -t ios --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
  ```
  把 `.p8` 放在 `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`。大约 15 秒完成。
- **"Upload succeeded" 不等于"通过"。** Apple 在处理阶段还会拒。轮询到 `VALID`，然后把上一个构建设为过期（`PATCH /v1/builds/{id}` 带 `{"expired": true}`）。
- **Watch 和小组件 target 需要图标**（`CFBundleIconName`），否则 Apple 会以错误 **90713** 拒绝上传。
- **ITMS-90062** 意思是"该版本已上架"：提升版本字符串。
- **ITMS-90863**（Apple Silicon 符号警告）在 Expo 应用里**很正常，不会导致拒绝**。别追着它跑。

### 重新提交的顺序
1. **两个版本不能同时在审。** 先取消现有的 `reviewSubmission`（`canceled=true`），等 CANCELING → COMPLETE。
2. 版本变成 `DEVELOPER_REJECTED`（可编辑）→ PATCH 版本字符串 → PATCH 构建关联。
3. ⚠️ **调包陷阱：** 取消之后紧接着附加构建会返回 409，如果脚本照样往下走，提交的就是**旧**构建。重试附加，并在提交前**核对已附加的构建**（`GET /appStoreVersions/{id}/build`）。
4. ⚠️ `POST reviewSubmissionItems` 在状态转换未完成时可能返回 409 `ENTITY_STATE_INVALID`。几秒后就能成功——做成可重试。

### 本地构建环境
- **如果 Xcode 在会话中途升级**，构建会以 "iOS X Platform Not Installed" 失败。解法：`xcodebuild -downloadPlatform iOS`（约 8.5 GB，不需要 sudo）加 `xcodebuild -runFirstLaunch`。当天早上编译成功过，不能证明环境现在还好。
- **Ruby 4.0 上的 CocoaPods：** `pod install` 以 `Unicode Normalization not appropriate for ASCII-8BIT` 挂掉。用 `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8` 运行。
- **Podfile 的 modular headers：** GoogleSignIn 9.2.0 需要给 `AppCheckCore`、`GoogleUtilities` 和 `RecaptchaInterop` 加 `:modular_headers => true`。
- **早于某个新 capability 的描述文件**会让本地构建失败。非交互模式的 EAS 不会更新凭证：要么交互式刷新，要么走 ASC API。
- **Apple Developer 的 capability 可以通过 API 开启**（不用进门户）：`POST /v1/bundleIdCapabilities`。不带 `settings` 会返回 409。
- **本地 Android 构建必须有 `ANDROID_HOME`**，否则 Gradle 会报 "SDK location not found"。
- **archive 编译期间绝不要改源文件** —— Metro 会打进一个写到一半的 bundle，应用启动就崩。
- **EAS 临时目录会无限增长**（我们那里到过 35 GB）。清理它；磁盘满了构建会以 "No space left" 失败。
- 失败的尝试导致构建号跳号是正常的。

### Play 发布错误
- **"此版本无法向现有用户提供……"** → 提高 version code，或者先走内部/封闭测试发布。
- **"此版本未添加或移除任何应用软件包。"** → AAB 没上传干净；检查 version code 后重新上传。
- **原生调试符号**必须放在 `native-debug-symbols.zip` 里，按 ABI 分目录（`armeabi-v7a/`、`arm64-v8a/`、`x86_64/`，各含 `libapp.so`），且不能有 `__MACOSX` 或 `.DS_Store` 条目。
- ⚠️ **目标 API 级别截止日期。** 对于错过截止日期的应用，Play 会阻止发布更新。盯着它。
- **AD_ID 的微妙之处：** Firebase Analytics 需要 manifest 里的权限和"使用"的声明；没有广告的应用两者都不该有。**规则是声明必须和 manifest 完全一致**——任一方向不一致都会卡住发布。

### 只在 standalone 构建里出现的崩溃
- **模拟器和 dev client 抓不到。** 用数据线在真机上测，配合 `devicectl --console`。
- 如果 `.env` 在 `.gitignore` 里，它永远不会进 EAS 的 archive：bundle 里变量为空，启动即崩。有个应用里*所有*构建都因此崩溃。
- 一个通过动态 `import()` 引入却没安装的原生模块，在开发环境里是隐形的（Metro 会提供它），到 standalone 就以 `RCTFatalException: Cannot find module` 崩溃。
- **Hermes 用 UTF-16 存字符串。** 在 bundle 里按 UTF-8 搜非 ASCII 字符串什么都搜不到：要按 UTF-16 验证。

---

## 商店注册 —— 一次性，而且是手动的

- **App Store Connect 的应用记录无法通过 API 创建。** 我们试过并确认了。用浏览器做。
- **在 Play Console 创建应用第一次同样是手动的。**
- **bundle ID 会永久绑定到该记录，无法更改。**
- **在 Play 上选"免费"是不可逆的**：上架之后无法改成付费。
- ⚠️ **非 ASCII 字符可能在注册时被丢掉。** 在个人 Apple 账号下，App Store 上显示的开发者名称就是你的法定姓名；我们的在注册时丢了变音符号。通过 App Store Connect → Business → Legal Entity 修改**不管用**：那个流程会把你拖进地址验证和 Paid Apps Agreement 的连锁，姓名本身存不下来。可行的路径是 Apple 支持 → "Membership & Account" → 法定姓名更正，需要身份验证。**注册时逐个字符核对拼写。**

## 执行本指南的 AI 助手的边界

- **绝不输入 Apple 或 Google 的密码和二次验证码。** App Store Connect 需要单独登录（开发者门户的会话不会延续）。流程是：人自己登录并确认，助手从那里接手 API 和控制台的操作。
- **浏览器文件上传上限是 10 MB；** 一个典型 `.aab` 超过 60 MB。让人自己上传，或者用 Play 服务账号加 `eas.json > submit.android` 自动化。
- **绝不在没有明确许可的情况下勾选声明或同意项。**

---

新的拒绝到来时，先找到根因，再往这里加一行。一份指南的价值，取决于上一次事故教会了它什么。
