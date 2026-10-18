---
name: media-luna-novelai
description: Write and debug NovelAI image prompts for the media-luna Koishi plugin — weight syntax (N::tag::, artist:name), the 512-token budget shared by main+character captions, character captions and positions, and the chat command flags (-t/-c/-r/-s/-d/-k/-n). Use when a NovelAI prompt or a `nai` chat command must be produced, translated from NovelAI metadata, trimmed, or diagnosed as blurry/garbled. 写、改写或排查 NovelAI 提示词与 media-luna 聊天命令时使用。
---

# media-luna × NovelAI 提示词与命令

目标读者是**要产出可直接粘贴的提示词或命令的代理**。NovelAI 的权重语法、token 预算、角色词机制有三处硬约束，
凭空写基本会踩；本文给出可直接使用的语法与量化边界。

## 一、先算 token，再写词

| 项 | 上限 | 计费方式 |
|---|---|---|
| 主提示词 + 所有角色提示词 | **512 token（共享一份预算）** | 合并计算 |
| 负面提示词 | 512 token | **单独**一份，不占主词 |
| V5 模型主词 | 1471 token | — |

这类「`权重::artist:名字::`、下划线标签、逗号分隔」的写法，实测约 **1.9～2.0 字符 = 1 token**。
经验边界（V4.5，实测同一 seed / 同参数）：

- 主词 + 角色词 = **960 字符 → 正常**
- 主词 + 角色词 = **1007 字符 → 整张糊**

所以按 **≤ 900 字符** 留余量。超预算的症状不是报错，而是**出图糊成一团**（角色层把主词挤掉，画面上只剩发色/服装类外观词）。

⚠️ 主词里还有**插件自动注入的前缀**。若该渠道的「提示词前缀」是「模型推荐质量词」，
会再加约 52 字符（如 `location, very aesthetic, masterpiece, no text`）——**计入 512**。
本插件该开关默认为「不注入」；写词前先确认，配方自带质量词时不要开启。

## 二、语法（直接可抄）

```
<权重>::<标签>::              权重分组，组内可含多个逗号分隔标签
2::Masterpiece, best quality::      整组 ×2.0
-6::artist_collaboration::          负权重（写在主词里也是合法用法）
0.75::artist:abara_heiki::          画师标签 + 权重
artist:名字                          画师引用；名字含下划线/括号要原样保留
{a} / [a]                            1.05× / 0.95× 强调（负面词里常见 {{{x}}} 叠加）
year 2025, rating:general             普通标签
```

- 权重语法里**逗号属于组内**：`1.4::artist:zuizi, artist:smilesmile1312::` 是一个组，两个标签。
- 标签名里的 `.` `_` `()` `@` `-` 都必须原样保留（`toto_(caaaaarrot)`、`na_tarapisu153`、`sigm@`）。
- 负面词里的 `{{{{bad anatomy}}}}` 是强调叠加，合法；不合法的是把主词权重语法误写进负面词。
- 「质量词组 + 场景词 + 画师堆」顺序不影响语义，但**超长时尾部会被丢弃**，把关键内容写前面。

## 三、角色词（character caption）

- 角色词与主词**共享 512**，不是各自一份。角色词 250 字符 ≈ 130 token，等于砍掉主词同等预算。
- 角色词只写**外观**：发色/瞳色/发型/体型/服装/配饰。
- 位置留空 = `use_coords: false`（角色词对整图生效，等同 NAI 元数据语义）；填 `A1`～`E5` 或 `0.25,0.7` 才启用空间约束。
- 想让外观生效又不想吃预算：把角色词压到 40～80 字符（如 `aqua eyes, ice blue hair, off-shoulder dress`）。

参考图本身的硬约束（插件已在本地处理，出错时照提示做）：

- 格式：**PNG 或 JPEG**（JPEG 会按文件内容识别并自动转 PNG）；WebP/GIF 不支持。
- 角色参考的**实际画布**由官方固定为 `1024×1536` / `1536×1024` / `1472×1472` 之一：插件按宽高比选最近的一档、等比缩放居中补黑边（不裁剪、不拉伸）。所以「参考图是方的还是竖的」由原图比例决定，改提示词改不了。
- 体积：base64 上限 **4,000,000 字符（约 2.9MB）**。插画类参考通常没问题，**摄影/高噪声参考的 PNG 常超限**（1472×1472 噪声图 ≈ 7MB → 9.9M 字符）——超限时插件本地直接报错，需要先裁剪或降低细节。

## 四、聊天命令

```
nai "<主词>" 2:3 -t 28 -c 6 -r 0.7 -s euler -d 162523019 -k "<角色词>" -n <负面词>
```

| Flag | 含义 | 取值 |
|---|---|---|
| `2:3` / `832x1216` | 分辨率、比例或**官方档位名** | 档位名：`小图竖幅/横幅/方形`（512×768 / 768×512 / 640×640）、`普通竖幅/横幅/方形`（832×1216 / 1216×832 / 1024×1024）、`大图竖幅/横幅/方形`（1024×1536 / 1536×1024 / 1472×1472）；边长须为 64 的倍数 |
| `透明` / `不透明` | 透明背景 | **仅 V5 系**实际下发（V4 系收了会 400）；默认不透明 |
| `-t` | steps | 1–64 |
| `-c` | CFG scale | 1–30 |
| `-r` | CFG Rescale | 0–1 |
| `-s` | sampler | `ea/euler/2m/2msde/2sa/sde/ddim` 或其编号 |
| `-d` | seed | 整数，仅本次生效 |
| `-k` | 角色词 | **可重复**；含空格要引号；可带 `@A1` |
| `-n` | 负面词 | **贪婪，吃掉行尾，必须放最后** |

尺寸默认值是官方九档的**第一档 512×768**；写不认识的尺寸不会静默换档，会明确提示并退回渠道配置。

### 接入配置与生成前确认（两种模式）

渠道配置里的「接入配置」两项：**配置 1 · windislands.com**（第三方中转，选它时 API 地址留空即用它）/ **配置 2 · 官方 NovelAI**。下面的两种模式与站点无关，只决定「要不要先估价确认」：

| 模式 | `confirmBefore` | 行为 |
|---|---|---|
| 模式 1（默认） | `on` | 出图前先回估价，60 秒内回复「确认」/`y` 才出图；其它内容或超时＝放弃且不消耗 |
| 模式 2 | `off` | 直接生成，出图后只报实际消耗 |

流程位置在**medialuna 原生的「已进入收集模式…」之后、「正在生成中…」之前**：

```
nai 1catgirl            → 已进入收集模式…（发「开始」触发生成）
开始                    → 估价 + 回复「确认」或「y」…（60 秒）
y                       → 正在生成中… → 出图 + 消耗回执
```

- 同群别人的发言既不算确认也不打断等待；确认只拦聊天命令，WebUI/控制台不受影响。
- 确认过价后，「正在生成中」那条提示**不再重复计费行**，整条流程只报一次价。
- 实现：core 的「生成前置守卫」（`mediaLuna.registerGenerateGuard`），命令层在真正出图前询问；不走管道中间件（`STOP` 会让管道走「成功但空输出」并补一句「生成完成，但没有输出」）。

**估价来源跟随接入配置**：

- **官方**：本地 Anlas 公式（面积 × 步数 + 参考附加，含 Opus 免费带）。官方 API **没有询价端点**——`/ai/*`、`/user/*`、`/oa/v1/*` 全查过，只有 `/ai/upscale` 的 `402 Not enough Anlas` 是扣费/拒绝信号；官方网页的「预计消耗」同样是客户端算的。
- **配置 1（windislands.com）**：`POST /api/billing` 的整单价（站长已对 Key 开放，自动启用；响应含 `cost/unitCost/totalCost/count/restrictedEligible/quoteToken`，参考图附加费按站方规则计入）+ `GET /api/connection` 的余额/基础价。报价拿不到时降级为「基础 N 积分/张 · 参考图附加费按站方结算」——**不自行推算**（原先的氛围编码报价与同形状实测兜底都已删除）。
- 生成后的回执 `消耗 N 积分 · 余 M 积分` 才是真实结算（基础价与实际可以差很多，实测 V4.5 + 1 张精准参考 = 14 积分）。

### 选项位置（已用 Koishi 解析器实测）

`-t/-c/-r/-s/-d/-k/-w` 放在提示词的**前面、中间、后面都行**，彼此顺序随意：

| 写法 | 结果 |
|---|---|
| `"词" -t 28 -c 6 -k "角色"` | ✓ |
| `-t 28 -c 6 "词" -k "角色"` | ✓ 选项在前 |
| `-k "角色" "词" -t 28` | ✓ 选项交错 |
| `"词" -n NEG -c 6` | ✗ 负面词变成 `NEG -c 6`，`-c` 失效 |
| `"词" -n=NEG -c 6` | ✓ 等号形式只吃一个 token（推荐） |
| `"词" --negative-prompt NEG -c 6` | ✗ 长名空格形式同样贪婪 |
| `"词" --negativePrompt NEG -c 6` | ✓ camelCase 长名只吃一个 token（可用但非规范写法） |
| `-t28 "词"` | ✗ 单破折号按字符拆成 `-t`(空) `-2` `-8` |
| `-t 28` / `-t=28` / `--steps 28` | ✓ 全部合法 |
| `"词 -t 28"` | ✓ 引号内不会被当选项 |

三条硬约束：

1. **`-n` 用空格形式（`-n 值`、`--negative-prompt 值`）时必须放最后**；要放中间就写 `-n=值`。
2. `-` 与值之间必须是空格或 `=`；且**值是紧邻的下一个 token**，`-t -c` 会把 `-c` 当成步数。
3. 主词含 `-6::` 这类连字符开头的内容必须用引号包住。

其他：

- 长名自动可用（kebab-case）：`--steps` `--scale` `--sampler` `--seed` `--cfg-rescale` `--reference-strength` `--character` `--negative-prompt`。
- 该渠道开了未知 Flag 检查，写错的 `-x` 直接报错，不会静默当正文。
- 选项字母与类型可在「命令选项覆盖」里改（改字母、把 `negativePrompt` 改成 `string` 版、或填 `-` 整条禁用）。

## 五、排查：图糊 / 图不对

按顺序查，不要先调采样器或步数（这两者对糊几乎无影响）：

1. **算 token**：主词 + 角色词字符数 ÷ 1.9。> 512 就是预算爆了 → 删画师标签或压缩角色词。
2. **看渠道前缀**：若「提示词前缀」为「模型推荐质量词」，额外 +52 字符。
3. **确认 seed / 参数真的送达**：生成回执会回显生效值与来源（本条/预设/记住/渠道）。
4. 精度判断用「同一 seed、只改一处」的 A/B，别跨 seed 比。

## 六、可复用的完整样例（已验证出图正常）

主词（656 字符）+ 角色词（252 字符）= 908 字符，未开前缀，安全落在 512 token 内：

```
nai "0.75::artist:abara_heiki::, 0.5::artist:au_(d_elete)::, 2::artist:ruoganzhao, artist:binggong_asylum::, 1.5::artist:min_(120716)::, 1.2::artist:ningen_mame, artist:toto_(caaaaarrot)::, 1.1::artist:ke-ta::, 0.8::artist:sheya::, 0.8::artist:na_tarapisu153, artist:sigm@::, 0.8::artist:onono_imoko::, 0.4::artist:mizumizuni::, 0.4::artist:tianliang_douhe_fangdongye, artist:omone_hokoma_agm::, 1.4::artist:zuizi, year_2025, solo_focus, long_sofa, indoors, bar, -6::artist_collaboration::, -3::multiple_views::, -3::simple_illustration::, 2::Masterpiece, best quality, amazing quality, highres, absurdres, photorealistic::, very aesthetic, masterpiece, no text" 2:3 -t 28 -c 6 -r 0.7 -s euler -d 162523019 -k "aqua eyes, multicolored eyes, ice blue hair, hair between eyes, medium hair, medium breasts, fang, snowflake_hair_ornament, hair ornament, hat ribbon, hairclip, ribbon, arm garter, thighhighs, halterneck, off-shoulder dress, high heel boots, leg garter" -n nsfw, lowres, worst quality, bad quality, jpeg artifacts, very displeasing, chromatic aberration, dithering, halftone, screentone, multiple views, logo, too many watermarks, bad anatomy, {{{{bad anatomy}}}}, bad hands, {{{deformed}}}, extra digits, extra limbs, {{missing fingers}}, {{{ugly}}}
```

该样例相对原始配方删掉了 `artist:smilesmile1312::`、`year_2024` 和结尾重复的 `year 2025`（共 47 字符），
正是这 47 字符把主词+角色词从 1007 压到 960、越过了边界。
