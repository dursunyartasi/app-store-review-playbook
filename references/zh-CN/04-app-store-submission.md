# 提交 App Store

构建已上传并且是 `VALID`。接下来是元数据、声明和审核备注——我们大部分拒绝其实是在这一步被决定的。

## 创建记录（一次性）

**App Store Connect 的应用记录无法通过 API 创建。** 我们试过并确认了；用浏览器做。bundle ID 会永久绑定到该记录。

之后的一切——附加构建、元数据、年龄分级、审核备注、提交——都可以通过 ASC API 完成。

## 截图

- **不存在 `APP_IPHONE_69` 这个截图类型。** API 接受的最大 iPhone 类型是 `APP_IPHONE_67`（1290×2796）。为 6.9 英寸设备渲染的 1320×2868 图片会被**拒绝**。上传 6.7 英寸的，让 Apple 自己放大。
- `whatsNew` **在首个版本上无法编辑** —— 409，"cannot be edited at this time"。它只存在于更新版本。

## 年龄分级

- 字段类型是混着的：有些是 BOOLEAN（`messagingAndChat`、`userGeneratedContent`、`advertising`），有些是 STRING 枚举（`contests`、`profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`）。类型填错返回 409，而错误信息会告诉你正确的集合。
- **Apple 在 2025 年改了分级档：12+ 已经不存在了。** 现在是 4+、9+、13+、16+、18+。
- 如实回答可能得到 4+；用 `ageRatingOverrideV2` 往上调（例如 `THIRTEEN_PLUS`）。
- **只要应用带有"结识他人 / 社交拓展"的成分，`matureOrSuggestiveThemes` 至少要声明为 `INFREQUENT_OR_MILD`。** 留空导致了一次 2.3.6 拒绝。

## App Privacy 声明

- **身份证号不属于 "Sensitive Info"。** Apple 的敏感数据类别涵盖种族、宗教、性取向、生物识别等；身份证号不在其中，所以正确的归类是 **"Other Data Types"**。
- **你自己存的银行信息算 "Collected"。** 只有当支付服务商持有、而你无法访问时，Apple 才豁免。
- ⚠️ **不要闭着眼睛点向导。** 它会按数据类型以不同高度渲染，所以重复点同一个位置产生过 "User ID 用于追踪：是" 这种完全错误的答案。每一项都截图核对最终状态。

## App Review 备注 —— 你会写下的杠杆最大的一段文字

我们有一次拒绝完全来自这个字段。Apple 的 4.2 拒绝 "small, or niche, set of users"，几乎逐字引用了我们自己写的句子：

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network."

**永远不要把应用描述成小众、niche、仅限邀请、封闭、私密、面向特定社群或非大众市场。** Apple 会把这理解为 Ad Hoc 分发，而不是 App Store。

改成这样写：面向所有人、免费、任何地方都能下载；任何精选或会员层级都是*可选*的。然后用编号步骤描述审核人员**不需要账号**就能走完的路径。如果应用确实没有这样的路径，提交前先造一条——这正是我们通过 4.2 的原因。

**该字段上限 4000 个字符。** 超出返回 409。

如果应用有不常见的目标端（Watch、小组件、某种设备特有的流程），在最上面放一段 "PLEASE READ FIRST"，写清楚登录步骤。

## 演示账号

勾选 "Sign-In Required" 并提供凭据。

- **先在真机上试一遍。** 有一次 2.1 拒绝，源于 Watch target 上从来就没通过的登录。
- **确保账号里有内容。** 有个应用里，17 个种子事件有 16 个已经过期，审核人员打开会看到一个空应用。准备一个幂等脚本把演示日期往前推，每次提交前跑一次。
- **验证墙会把审核人员困住。** 如果注册但未验证的人什么都看不到，应用看起来就是封闭的。让访客能浏览，只在写入操作时要求验证。
- **通过之后关掉演示账号。** 它的密码存在 App Store Connect 里。

## 法律链接

条款和隐私链接必须**可点击**，用应用内浏览器打开而不是把人甩去 Safari，而且**登录页**也要有，不能只在注册页。不可点的纯文本导致了一次 2.1(a) 拒绝：审核人员读不到条款，仅凭这一点就拒了。

## 如果应用免费但在别处卖东西

3.1.1 是免费应用和 B2B 应用的陷阱。**把所有价格、套餐名称、额度计数器、付费墙、升级按钮和站外购买链接全部去掉。** 光是一个套餐名称就足以毁掉一个构建。

3.1.3(f) "Free Stand-alone Apps" 这条论证**单独用在我们身上没成功。** 薄弱环节是一个公开的注册页面——它读起来就是面向消费者的自助销售，和 3.1.3(c) 的 "only sold directly by you to organizations" 直接矛盾。我们删掉了注册页，只保留登录。

## 提交，以及被拒后重新提交

顺序很重要。弄错会悄无声息地提交错误的二进制包。

1. **两个版本不能同时在审。** 先取消现有的 `reviewSubmission`（`canceled=true`），等 CANCELING → COMPLETE。
2. 版本会变成 `DEVELOPER_REJECTED` 并可编辑。先 PATCH 版本字符串，再 PATCH 构建关联。
3. ⚠️ **调包陷阱。** 取消之后紧接着的附加构建调用会返回 409。如果你的脚本照样往下走，它就会提交**旧的**构建。重试附加，然后在提交前用 `GET /appStoreVersions/{id}/build` **核对**。我们就这样发过一次错误的构建。
4. ⚠️ `POST reviewSubmissionItems` 在状态转换尚未完成时可能返回 409 `ENTITY_STATE_INVALID`。几秒后就能成功——把这一步做成可重试。

发布类型默认是**手动**：通过之后仍然需要有人去点发布。

## 做好不止一轮的准备

有个应用连续被拒四次，另一个三次。修好一个可能暴露下一个，而在一个地方的修改可能在另一个地方制造问题。**每次修完，重新通读 `05-rejections.md` 的整份清单**，而不只是你改动的那一条。
