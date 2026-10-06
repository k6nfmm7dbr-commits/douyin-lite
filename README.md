<img src="icon.png" width="96" align="right" alt="Douyin Lite">

# 抖音瘦身 · Douyin Lite

[![validate](https://github.com/k6nfmm7dbr-commits/douyin-lite/actions/workflows/validate.yml/badge.svg)](https://github.com/k6nfmm7dbr-commits/douyin-lite/actions/workflows/validate.yml)
![clients](https://img.shields.io/badge/clients-Egern%20%7C%20Surge%20%7C%20Loon%20%7C%20Stash%20%7C%20Shadowrocket-blue)
![license](https://img.shields.io/badge/license-MIT-green)

一个去广告 / 降内存的代理模块：拦截抖音的**穿山甲 / 巨量引擎广告请求、埋点和信息流广告接口**，
减少广告视频的下载与解码。

支持 **Egern**（原生 YAML 模块）、**Surge / Stash / Shadowrocket**（`.sgmodule`）、
**Loon**（`.plugin`）。任意一份都能用 [Script-Hub](https://github.com/Script-Hub-Org/Script-Hub) 互相转换。

---

## ⚠️ 先说结论，避免错误期待

代理层（Egern / Surge / Loon 这类）**无法直接降低抖音的内存占用**。
抖音的内存大头全在 App 进程内部，模块碰不到：

| 内存大头 | 量级 | 代理层能否触及 |
|---|---|---|
| 视频解码器实例 + 解码帧缓冲 | 大 | ⚠️ 只能减少"要解码的视频条数" |
| 预加载缓冲（抖音会预取后面几条） | 大 | ❌ 不能 |
| 直播 / 商城 / 团购页面的 WebView | 中 | ❌ 不能 |
| 视频文件磁盘缓存 | 中 | ⚠️ 只能减少"要缓存的东西" |
| 埋点 / 上报队列 | 小 | ✅ 能 |
| **信息流广告视频** | 中 | ✅ **能** |

不过抖音这边有一个比微信更实在的点：**一条信息流广告就是一段完整视频**，
它和正常视频一样要占解码器 + 帧缓冲 + 预加载槽位。把它拦掉，省下的是**实打实的一整段视频内存**，
而不是微信那边"少几张图"。

所以：**这个模块能拿到比微信那边更明显的收益，但它依然不是银弹。**
另一半在 [App 内设置](#app-内设置真正有效的另一半) 那一节。

---

## 核心难点：为什么不能拦域名

**抖音信息流广告的视频素材，和正常视频共用同一个 CDN**（`douyinvod.com` / `douyinpic.com`）。
域名层面完全没法区分广告和正常内容 —— 拦了 CDN 抖音直接不能看。

唯一的区分点是**请求路径**：

```
https://aweme.snssdk.com/api/ad/...
https://api5-normal-m.amemv.com/api/ad/...
https://zlink.ugsdk.cn/ad/...
```

而要看路径就必须 MITM。这就是为什么进阶版需要证书，而基础版只能拦广告 SDK 和埋点。

---

## 安装

### 版本选择

| 版本 | 说明 | 需要 MITM / 证书？ |
|---|---|---|
| **基础版**（推荐先用） | 穿山甲 / 巨量引擎 / 埋点域名拦截 | ❌ 不需要 |
| **进阶版 Plus** | 基础版 + 信息流广告接口拦截（`/api/ad/`） | ✅ 需要 |

**两个版本选一个用，不要同时装。**

### 导入链接

| 客户端 | 基础版 | 进阶版 |
|---|---|---|
| **Egern** | [douyin-lite.yaml](https://raw.githubusercontent.com/k6nfmm7dbr-commits/douyin-lite/main/egern/douyin-lite.yaml) | [douyin-lite-plus.yaml](https://raw.githubusercontent.com/k6nfmm7dbr-commits/douyin-lite/main/egern/douyin-lite-plus.yaml) |
| **Surge / Stash / Shadowrocket** | [douyin-lite.sgmodule](https://raw.githubusercontent.com/k6nfmm7dbr-commits/douyin-lite/main/surge/douyin-lite.sgmodule) | [douyin-lite-plus.sgmodule](https://raw.githubusercontent.com/k6nfmm7dbr-commits/douyin-lite/main/surge/douyin-lite-plus.sgmodule) |
| **Loon** | [douyin-lite.plugin](https://raw.githubusercontent.com/k6nfmm7dbr-commits/douyin-lite/main/loon/douyin-lite.plugin) | 用 Script-Hub 转换 Surge 进阶版 ↓ |

- **Egern**：工具 → 模块 → 右上角 `+` → 粘贴上面的链接
- **Surge**：首页 → 模块 → 安装新模块 → 粘贴链接
- **Loon**：配置 → 插件 → `+` → 粘贴链接
- **不想托管任何文件**：把 [`egern/snippet-profile.yaml`](egern/snippet-profile.yaml) 里的
  `rules` 直接粘进 Egern 主配置 `Profile.yaml` 即可

### 转成别的客户端

本项目所有版本共用同一份规则，任意一份都能互转：

> 在 Surge / Egern 里打开 **`script.hub`**（或访问 <https://scripthub.vercel.app>）
> → 来源类型选「Surge 模块」，粘贴上面的 raw 链接 → 目标类型选你要的客户端。

**Loon 的进阶版**就是用这个方法把 `douyin-lite-plus.sgmodule` 转出来的。

---

## 思路：为什么是这几刀

代理模块唯一能控制的变量是**网络请求**，逻辑链条还是那一条：

```
少请求 → 少下载 → 少解码 → 少缓存 → 内存峰值下降
```

### 1. 穿山甲 / Pangle 广告 SDK —— 零风险

字节自家的广告 SDK。`pangolin` 这个关键词一条就能覆盖整个家族：
`pangolin-sdk-toutiao{,-b,1}.com`、`pangolin-dsp-toutiao{,-b}.com`、
`pglstatp-toutiao{,-b}.com`、`pangle.cn`、`pangle.io`。

### 2. 巨量引擎（字节广告投放平台）—— 零风险

`oceanengine.com` / `oceanengine.cn`。抖音 App 自己不需要访问这些域。

### 3. 广告投放 / 购物广告 / 试玩广告 —— 零风险

`ad.toutiao.com`、`adshare.toutiao.com`、`shoppingads.cn`、
`panplayable-toutiao`（试玩广告），以及字节 API 域里带 `ads` 的广告子域
（`DOMAIN-REGEX:^.*ads.*-normal\.zijieapi\.com$`）。

### 4. 埋点 / 上报 —— 收益小，顺手

`extlog.snssdk.com`、`mcs.snssdk.com`、`sdfp.snssdk.com`、`bds.snssdk.com`、
`ctobsnssdk.com`、`volceapplog.com`。省一点网络与后台线程，对内存帮助很小。

### 5. 信息流广告接口 —— 进阶版，需 MITM，**本模块的核心收益**

`/api/ad/` 返回空 JSON，App 拿到的就是"这次没有广告"，于是**整条广告视频不会下载、不会解码**。

用返回 `{}` 而不是直接 reject：reject 会断开连接，App 可能当成网络错误去重试，
反而多出请求。给个合法空响应最干净。

### 故意没做的事

| 想法 | 为什么不做 |
|---|---|
| 拦视频 CDN（`douyinvod.com` / `douyinpic.com` / `douyinstatic.com`） | 广告和正常内容共用，拦了抖音直接不能看 |
| 拦 API 主域（`amemv.com` / `snssdk.com` / `zijieapi.com`） | 抖音直接打不开。只拦其中明确带 `ads` 的子域 |
| 拦直播 / 商城 / 团购相关域 | 对应功能直接坏 |
| 用 jq 脚本删侧边栏、常用功能、tab 配置 | 那是 **UI 精简**，不是省内存；而且依赖会随版本变的接口结构，很容易失效。想要 UI 精简请用社区那类「侧边栏精简」模块，别混在一起 |
| 拦视频预加载 | 抖音靠预加载保证滑动流畅，拦了就是卡，得不偿失 |

宁可少做几刀，也不要做出一个"用着用着就坏"的模块。

---

## 文件结构

```
.
├── egern/
│   ├── douyin-lite.yaml          基础版 · Egern 原生模块
│   ├── douyin-lite-plus.yaml     进阶版 · Egern 原生模块
│   └── snippet-profile.yaml      可直接粘进 Egern 主配置的片段
├── surge/
│   ├── douyin-lite.sgmodule      基础版 · Surge 格式
│   └── douyin-lite-plus.sgmodule 进阶版 · Surge 格式
├── loon/
│   └── douyin-lite.plugin        基础版 · Loon 插件
├── tools/check.py                一致性校验（CI 用）
└── icon.png
```

`tools/check.py` 会校验所有客户端版本的**规则集完全一致**、元信息齐全、
基础版不含 MITM / 进阶版三件套齐全。改规则时只需保证各处同步，CI 会兜底。

```bash
pip install pyyaml && python3 tools/check.py
```

---

## ⚠️ 三个必须注意的点

### 1. 规则顺序决定生死

Egern / Surge / Loon 的规则都是**从上到下匹配第一条命中的**。
如果你主配置里有这类宽泛规则且排在前面：

```yaml
- domain_suffix:
    match: "snssdk.com"
    policy: DIRECT          # ← 广告域名也匹配 snssdk.com，会被它抢先放行
```

那么本模块的 `REJECT` **完全不会生效**。请确保本模块规则排在它之前。

**验证**：请求记录 → 刷一下抖音 → 看 `pglstatp-toutiao.com`、`oceanengine.com` 是否显示 `REJECT`。

### 2. 进阶版会解密抖音的主 API 域 —— 这是本项目风险最大的一段

`/api/ad/` 挂在抖音的**主 API 域**上，要拦它就必须 MITM 这些域。
万一抖音启用了**证书固定（cert pinning）**，MITM 会让这些域直接连不上，
表现是**抖音打不开 / 一直转圈**。

真遇到就**删掉 `[MITM]` 段和 `[Map Local]` 段**，回到基础版 ——
基础版的域名拦截已经能拿到大部分收益。

### 3. MITM 没装好不会报错，只会静默失效
证书没安装 / 信任时，Map Local 规则不会生效，但**也不会报错**（域名拦截部分照常工作），
容易误以为"开了没用"。

检查路径（以 Egern 为例）：工具 → MITM → 确认 `aweme.snssdk.com` 在列表里，且证书已安装并信任：

- iOS：设置 → 通用 → VPN 与设备管理（安装描述文件）
- iOS：设置 → 通用 → 关于本机 → 证书信任设置（打开开关）

### 4. 广告还在？

抖音的广告接口路径会随版本调整。如果进阶版开了之后信息流里还能刷到广告：

1. 打开请求记录，刷到广告，找那条**刚发出的广告请求**，看它的真实路径；
2. 如果路径不是 `/api/ad/`，照着 `map_locals` 里已有的格式加一条即可。

同理，如果发现新的广告域名，也欢迎提 issue 补充。

---

## App 内设置：真正有效的另一半

模块只能省"广告和埋点"那部分。**下面这些才是抖音内存的大头**，效果比模块大得多。
（抖音设置路径随版本变化较快，以下是大致位置。）

### 1. 清缓存 —— 最立竿见影

> 我 → 右上角 `≡` → 设置 → 通用设置 → 清理缓存

抖音会把看过的视频缓存在本地（占用经常是几个 GB）。清完之后内存和存储都会明显改善。

### 2. 播放设置

> 我 → 右上角 `≡` → 设置 → 通用设置 → 播放设置

- 关掉「移动网络下自动播放」，减少移动网络下的预加载。
- 如果有关闭「自动保存到相册」的选项，也关掉。

### 3. 系统层

- **iOS 设置 → 通用 → 后台 App 刷新 → 关闭抖音**。
  抖音的消息推送走 APNs，关掉不影响收消息，但能减少后台驻留和预加载。
- **定期彻底退出抖音**（上滑卡片杀掉）再打开。
  这一步专门清掉**视频解码器实例、预加载缓冲和直播/商城 WebView 的残留** ——
  模块完全碰不到这块，但连续刷几小时后它往往是内存涨得最凶的部分。

### 4. 别把直播挂在后台

抖音的直播间是最吃内存的场景（视频解码 + 弹幕 + 礼物动画 + 商城挂载）。
不看了就退出直播间，别只是划走。

---

## 预期效果 & 怎么验证

**合理预期**：

- 信息流广告不再出现（进阶版）→ 每条广告省下一整段视频的下载、解码和缓存；
- 穿山甲 / 巨量引擎相关请求被拒 → 省流量；
- 但**抖音的常驻内存和正常视频的预加载缓冲不会因为这个模块下降**。

**怎么验证**：

1. 请求记录 → 刷抖音 → 确认 `pglstatp-toutiao.com`、`oceanengine.com` 显示 `REJECT`；
2. 进阶版：刷信息流 → 请求记录里 `aweme.snssdk.com/api/ad/` 应显示为 Map Local 命中；
3. 对比开启前后「设置 → 通用 → iPhone 储存空间」里抖音占用的增长速度；
4. 主观指标：刷久了之后切回抖音是否重新加载、滑动是否卡顿。

如果都没变化，大概率是**规则顺序问题**（见上一节第 1 点）。

---

## 风险清单

| 风险 | 说明 | 缓解 |
|---|---|---|
| 规则不生效 | 被更靠前的宽泛规则抢先命中 | 用请求记录确认，调整顺序 |
| **MITM 抖音主 API 域** | 若存在证书固定，抖音可能打不开 | **删掉 `[MITM]` + `[Map Local]` 段回到基础版** |
| `oceanengine.com` 被拦 | 会连带影响**巨量引擎广告投放后台网页**（如果你自己是广告主） | 注释掉那两行 |
| `pangolin` 关键词拦截 | 关键词匹配面较广 | 出问题就注释掉，改用精确域名列表 |
| 埋点拦截 | 个别上报失败重试可能略增后台活动 | 注释掉对应那几行 |
| 广告位留白 | 个别位置可能显示空白而非消失 | 可接受 |

任何一条出问题，都是注释掉对应那行的事，不用重装。
**回滚**：关掉模块开关即可，没有改任何持久状态。

---

## 域名清单来源

模块里的域名不是拍脑袋写的，逐条对照了社区长期维护的规则集做了交叉验证：

| 来源 | 提供的条目 |
|---|---|
| [LoonKissSurge / 香港抖音去广告.beta](https://github.com/QingRex/LoonKissSurge) | `/api/ad/` 广告接口路径（`aweme.snssdk.com`、`api5-normal-*.amemv.com`、`zlink.ugsdk.cn`）、MITM hostname 写法 |
| [LoonKissSurge / 广告平台拦截器.beta](https://github.com/QingRex/LoonKissSurge) | `pangle.io`、`pangolin`、`panplayable-toutiao`、`ctobsnssdk.com`、`extlog.snssdk.com`、`mcs.snssdk.com`、`bds.snssdk.com`、`*-normal.zijieapi.com` 的 ads 判定 |
| [blackmatrix7 / Advertising](https://github.com/blackmatrix7/ios_rule_script) | `ad.toutiao.com`、`adshare.toutiao.com`、`pglstatp-toutiao`、`sdfp.snssdk.com` |
| [blackmatrix7 / ByteDance](https://github.com/blackmatrix7/ios_rule_script) | `pangle.cn`、`oceanengine.com` / `.cn`、`shoppingads.cn`、`volceapplog.com`、穿山甲域名全家族 |

社区用精确域名的地方（如 `pglstatp-toutiao` 而不是整个 `toutiao.com`），本模块也保持精确，
避免误伤面被无谓放大。

### 为什么不做成脚本版

抖音这边的广告是**接口下发 + 共用 CDN 的普通视频**，没有任何一处需要解析或改写 HTML/JSON 结构：

- 域名层能拦的，规则就够了；
- 域名层拦不了的（`/api/ad/`），Map Local 返回 `{}` 就够了，不需要 jq 去改结构。

jq 改写（删侧边栏、改 tab 配置那类）属于 **UI 精简**，依赖会随版本变的接口结构，
跟"降内存"是两件事，混在一起只会让模块更容易坏。所以本模块保持**纯声明式**：
零脚本、零 jq。

---

## License

[MIT](LICENSE)
