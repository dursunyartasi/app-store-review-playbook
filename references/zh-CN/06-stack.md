# 指南背后的基础设施

[拒绝案例库](05-rejections.md) 讲的是怎么过审。这份文件讲的是底下跑着什么：这些应用发布所依赖的基础设施，以及我们运维它时遇到的故障。

规则和指南一样：只写真实发生过的。这是**我们在用的**，不是在断言你应该用什么。

---

## 我们跑的是什么

| 层 | 选择 | 原因 |
|---|---|---|
| 移动端 | **Expo / React Native**（SDK 57） | 一套代码覆盖两个商店；EAS 或完全本地构建 |
| Web / API | **Next.js** | 两端同一套 TypeScript |
| 数据库 | **PostgreSQL**，自托管 | 成本可预测，没有按行计费的意外 |
| ORM | **Prisma** | 迁移在碰到生产之前可以先审阅 |
| 文件 | **MinIO**（S3 兼容）或 Cloudflare R2 | 自托管对象存储，没有出网流量账单 |
| 托管 | 普通 VPS 上的 **Coolify** | 自托管 PaaS：git 部署、TLS 和容器，不按服务计费 |
| 邮件 | **Brevo** 免费额度，走 SMTP | 每天 300 封免费，OTP 和通知能撑很久 |
| 支付（土耳其） | **iyzico** | 本地银行卡和分期，Stripe 在当地覆盖不了 |

有一个应用跑在 Supabase 上而不是自托管 PostgreSQL。两者都能用；下面会标出哪些经验是 Supabase 特有的。

---

## Coolify 与部署

Coolify 是跑在你自己 VPS 上的自托管 PaaS。它省掉了按服务计费的托管账单，同时把托管平台本来会替你吸收的运维故障交还给你。

### 磁盘压力才是你真正会遇到的故障

一旦服务器磁盘超过大约 **80%**，即使构建本身成功了，部署也会在导出镜像层的阶段失败。Coolify 把它显示成 `exit code 255` 或一个笼统的 `DeploymentException`——**真正的原因被藏起来了。** 导出大概需要 20 GB 空闲空间。

```bash
docker system df           # 先看
docker builder prune -af   # 构建缓存是可以安全删掉的那部分
```

构建缓存可以放心清理（下次构建稍慢一点）。镜像大多是被引用着的，清理它们释放不了多少。**千万别碰 volume——那是你的应用数据。** 有一次事故里，这一步把磁盘从 92% 降到 83%，释放了 7.6 GB；重试之后部署就成功了。

同样的磁盘压力还会表现为构建中途辅助容器死掉时的临时 `No such container: <uuid>`。内存压力会产生一模一样的症状，所以两边都要查。

### 其他值得知道的部署行为
- **一次部署会重建 compose 里的所有服务**，不只是改动的那个——包括你的数据库容器，而它的**名字会变**。任何绑定容器名的东西都会断：每次部署后重新解析。
- **一次部署大约 200 到 300 秒。** 轮询到新容器出现并且返回 HTTP 200 为止；不要凭触发调用就认定成功。
- **第一次尝试可能毫无理由地失败**在 compose 阶段。重试通常就好了，生产不会挂。
- **部署默认不由 webhook 触发**——是手动或 API 操作。
- 如果你的 VPS 在 **Cloudflare 后面**，注意 `urllib` 的默认 user agent 是被拦的。给自己的 API 写脚本时用 curl 或设一个浏览器 user agent。

### Postgres 备注
- **Supabase / PostgREST：** 新建的表明明存在，却返回 `PGRST205 "Could not find the table in schema cache"`。是 REST 缓存旧了。解法：`NOTIFY pgrst, 'reload schema'`。
- **Realtime 需要 `wal_level=logical`。** 在默认的 `replica` 下，`postgres_changes` 订阅得好好的，然后一个事件都不推——一个看起来像客户端 bug 的静默故障。改这个需要重启容器，所以安排一个维护窗口。

---

## 免费额度上的邮件，以及差点毁掉一切的 DNS 陷阱

Brevo 的免费额度（每天 300 封）足以支撑 OTP、密码重置和通知很长一段时间。把应用指向 `smtp-relay.brevo.com:587`。

要让邮件进收件箱而不是垃圾箱，域名必须在 Brevo 里显示为 **Authenticated**，这需要：
- **DKIM** —— Brevo 给的那两条 CNAME 记录
- **DMARC** —— 从 `p=none` 开始
- **SPF** —— `include:spf.brevo.com`
- Brevo 的验证 TXT 记录

### ⚠️ SPF 陷阱
我们在同一个域名上开了 Cloudflare Email Routing 来*接收*邮件。Cloudflare 提议"补上缺失的记录"，发现该域名已经有一条 Brevo 的 SPF 记录，于是建议通过**删掉 Brevo 那条**来解决冲突。

接受这个建议，会让应用发出的每一封邮件——OTP、通知、密码重置——失去身份验证，全部掉进垃圾箱。正确做法是把两个 include 合并成**一条**记录：

```
v=spf1 include:spf.brevo.com include:_spf.mx.cloudflare.net ~all
```

**一个域名必须恰好有一条 SPF 记录。** 多于一条违反 RFC，并且会让所有发信失效。用 `dig` 验证，别信面板。

### MX 陷阱，以及它为什么是个商店问题
同一个域名**根本没有 MX 记录**。它能发，不能收。我们公布出去的审核联系邮箱，没有任何人能收到。

这不只是一个邮件 bug。App Store **Guideline 1.2** 要求有一条可用的内容举报途径，而我们自己的规则承诺三个工作日内回复。一个默默丢弃邮件的地址，既是没兑现的承诺，也是审核风险。**如果你在商店信息或应用内规则里公布了联系地址，给它发一封测试邮件。**

还有一点：Brevo 可能会把发信限制在一个 IP 白名单里。把你的开发机和服务器都加进去，否则本地测试通过而生产邮件死掉。

---

## 移动构建备注

完整的构建与上传陷阱在[案例库](05-rejections.md)里。这里是它们背后的基础设施决策：

- **迭代阶段本地构建胜过 EAS 远程构建。** 远程队列会排满，而非交互模式的 EAS 不会更新凭证——所以一个早于新 capability 的描述文件会把你卡死且无路可走。本地 `xcodebuild` 加 `xcrun altool` 就是逃生通道。
- **从 EAS 的角度想 `.env`。** 被忽略的 `.env` 永远进不了 archive，结果是变量为空，以及只在 standalone 构建里出现的启动崩溃。
- **本地 Android 构建需要 `ANDROID_HOME`**，否则 Gradle 会报 "SDK location not found"。
- **用服务账号把 Play 上传自动化**（`eas.json > submit.android`）。手动传 `.aab` 是最久保持手动的那一步，而且浏览器自动化帮不上忙：文件远超任何上传大小限制。

---

## VPS 从哪来

Coolify 需要一台有 root 权限的普通 VPS，不需要托管平台。任何提供 Docker 和公网 IP 的服务商都可以。按我们跑的规模来估：应用本身一个小实例就够，但**给磁盘留出比你觉得需要的更多空间**，因为上面那个导出镜像层的故障是磁盘问题，不是 CPU 问题。在镜像之外预留 20 GB 余量。

我们的跑在 Hostinger 上。**推荐链接 — [hostinger.com](https://www.hostinger.com/tr?REFERRALCODE=KAWDURSUNLTO)** — 使用它作者会得到一笔佣金，你会得到折扣。这不是必需的：Coolify 能跑在任何提供 Docker 和 root 权限的服务商上，本指南没有任何内容依赖于主机商。

---

## 这些和审核的关系

案例库里有好几个商店拒绝，其实是披着 guideline 编号的基础设施问题：

| 看起来是 | 实际上是 |
|---|---|
| Guideline 1.2，没有举报内容的途径 | 一个公布出去却没有 MX 记录的联系地址 |
| Play 提交被拒 | 声明的隐私政策 URL 返回 404 |
| 2.1 App Completeness，"应用启动即崩" | `.env` 从来没进过构建 |
| 2.1，"我们访问不了这个功能" | 生产环境里一个关着的功能开关 |

在怪审核人员之前，先确认对方够不到的那个东西，从你机器之外是不是真的够得到。
