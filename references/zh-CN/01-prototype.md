# 从零到一个能跑起来的原型

目标：一个跑在对方手机上的应用，并且从一开始就满足 `05-rejections.md` 里的商店要求，而不是事后再补。

## 写第一个文件之前要定的事

**managed 还是 bare workflow。** managed 把 `ios/` 和 `android/` 作为生成目录；bare 把它们提交进仓库。bare 给你原生控制权，代价是：**`app.json` 不再是唯一事实来源。** 权限用途说明不再自动同步到 `Info.plist`，版本号改为来自 `Info.plist` 的 `CFBundleShortVersionString` 加 pbxproj 的 `MARKETING_VERSION`。这两件事都让我们被拒过。除非某个原生模块逼你，否则从 managed 开始。

**Bundle identifier。** 现在就定好，而且要确定——App Store Connect 记录一旦创建，**bundle ID 就是永久的**。用你自己控制的域名做反向 DNS。

**Apple 会显示的名字。** 在个人开发者账号下，App Store 上显示的开发者名称就是你注册时的法定姓名。非 ASCII 字符在注册时可能被悄悄丢掉（我们的土耳其语变音符号就没了），而 App Store Connect 的自助修改**不管用**——那个流程会把你拖进地址验证和 Paid Apps Agreement 的连锁里，姓名本身根本存不下来。真要改，得走带身份验证的支持工单。**注册时逐个字符核对拼写。**

## 搭架子

```bash
npx create-expo-app@latest my-app
cd my-app
npx expo start            # 用 Expo Go 扫码
```

在你加入原生模块或需要签名构建之前，Expo Go 就够了。之后就需要 development build 或真正的 archive。

## 现在就把商店要求接进去

这些东西第一天做很便宜，被拒之后再做很贵。

**如果用户能登录——账号删除（5.1.1(v)）。** 必须能在应用内找到，立即且永久。不能是停用、不能有冷静期、不能"发邮件给客服来删"、不能跳转到网页。重新要一次密码，给一个破坏性操作的确认，列出会删掉什么，第三方授权也要在对方平台上一并撤销。

**如果用户能发布内容——1.2 的四项要求。** 每条消息、帖子和评论旁边都要有可见的入口（一个"⋯"按钮；长按手势对审核人员是隐形的，我们因此被拒），双向生效的拉黑，**所有**写入接口上的内容过滤，以及陌生人发送第二条私信之前的同意步骤。

**法律链接必须可点（2.1(a)）。** "注册即代表你同意条款"如果是纯文本，就是一次拒绝。做成真正的链接，用应用内浏览器打开而不是把人甩去 Safari，而且登录页也要有，不能只放在注册页。

**权限。** 不用的一律别要。在打开选择器之前申请完整相册权限，让我们被拒过一次——现代 iOS 选择器不需要任何权限就能返回一张照片。前置引导页不能用有倾向性的按钮文案："继续"，而不是"使用我的位置"。

**一个真能收到邮件的联系地址。** 如果你在商店信息或应用内规则里公布了一个地址，这个域名就需要 MX 记录。见 `06-stack.md` 里的 MX 陷阱——我们那个域名能发不能收，公布出去的审核联系邮箱其实没人收得到。

## 环境变量

```
.env            → 照常写进 .gitignore
.easignore      → EAS 读的是这个，而且它会取代 .gitignore
```

**被忽略的 `.env` 永远不会进入 EAS 的 archive。** 打包出来的 bundle 变量全空，应用一启动就崩——而且**只在** standalone 构建里崩，所以模拟器和 dev client 看起来一切正常。我们有个应用在找到原因之前，所有构建都因此崩溃。要么在 EAS 上配置环境变量，要么确认 `.easignore` 没有排除掉构建需要的东西。

## 装到真机上

模拟器证明不了应用能跑。只在 standalone 出现的崩溃，正是会送到审核人员手里的那一类：

```bash
npx expo export --platform ios --output-dir /tmp/exportcheck   # 提前抓出 import 错误
```

然后用数据线编译安装，用 `devicectl --console` 看日志。一个通过动态 `import()` 引入却没安装的原生模块，在开发环境里是隐形的（Metro 会提供它），到了 standalone 就以 `RCTFatalException: Cannot find module` 崩溃。

## 继续往下之前

跑 `npx tsc --noEmit` 和你的测试，都要干净。从这里往后，每一轮构建要花 5 到 40 分钟；一旦进了审核，就是以天计。

下一步：`02-testflight-ios.md` 或 `03-google-play.md`。
