# 资源包

资源包是**纯 JSON 的「识别与释义」数据**，把「这个目录是什么、能不能自动重建、
删掉有什么影响、什么情况该留着」这段说明，从代码里拆出来变成可替换、可下载、
可由使用者自己写的文件。

内置知识库（`config/filetypes.json`、`config/targets.json`）覆盖通用情况；
资源包补的是长尾：游戏中文名、Windows 系统项、各软件的缓存目录。

## 目录约定

| 位置 | 谁写的 | 会不会进版本库 |
| --- | --- | --- |
| `packs/` | 本仓库随源码分发 | 会 |
| `config/packs-user/` | 用户导入，或界面里「从 GitHub 检查更新」拉下来的 | 不会（见 `.gitignore`） |

同名 `id` 时用户包覆盖内置包 —— 这也是更新机制：远端拉下来的包写进
`config/packs-user/`，因此不会改动仓库里的文件。

## 文件格式

```json
{
  "id": "my-pack",
  "name": "我的资源包",
  "version": "1.0.0",
  "type": "custom",
  "description": "一句话说明这个包是干什么的",
  "author": "你的名字",
  "license": "MIT",
  "updated": "2026-10-12",
  "entries": [
    {
      "id": "some-entry",
      "title": "显示用的名字",
      "category": "临时文件",
      "risk": "低",
      "cleanable": true,
      "only": "dir",
      "match": {
        "any": ["\\appdata\\local\\someapp\\cache"],
        "all": []
      },
      "what": "它是什么。",
      "rebuild": "能不能自动重建。",
      "effect": "删掉有什么影响。",
      "keep": "什么情况建议保留。"
    }
  ]
}
```

### 字段说明

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `id`（包） | 是 | 2~64 个字符，只允许字母、数字、`.`、`_`、`-` |
| `name` | 是 | 显示名 |
| `type` | 否 | `games` / `system` / `apps` / `custom`，只影响界面上的分类标签 |
| `entries` | 是 | 非空数组，上限 5000 条 |
| `entry.id` | 否 | 包内唯一即可，用于排查问题 |
| `entry.title` | 是 | 中文名或显示名 |
| `entry.match` | 是 | 至少给出 `any` 或 `all` 之一 |
| `entry.only` | 否 | `dir` 或 `file`，用来限定只匹配目录或只匹配文件 |
| `entry.category` | 否 | 归类，如 `临时文件`、`显卡缓存`、`游戏` |
| `entry.risk` / `cleanable` | 否 | **仅在项目未被内置规则识别时才生效**，见下方「安全边界」 |
| `entry.what` / `rebuild` / `effect` / `keep` | 否 | 四段说明，与卡片上的「?」弹窗一一对应 |

### 匹配语义

匹配对象是**文件/目录名**与**完整路径**（都转成小写、分隔符统一为 `\`），
两者任一命中即算命中：

* 不含前缀的字符串 → 子串匹配。`"steamapps\\common"` 会命中
  `E:\SteamLibrary\steamapps\common\GameName`。
* 前缀 `re:` → 后面的部分按正则处理。例如 `"re:^game[0-9]+\\.exe$"`。
  正则写错时这一条按「不命中」处理，不会让扫描中断。
* `match.any` → 列表里**任一**项命中即可。
* `match.all` → 列表里**全部**项都必须命中，用来加范围约束。
* 同时给出 `any` 与 `all` 时，两者都要满足。

命中的条目**超过一条时，取匹配串最长的那条** —— 越长的关键字越具体。
所以 `steamapps\shadercache` 会胜过 `\shadercache`。

## 安全边界

资源包**只能补空缺，不能改写结论**：

* 内置知识库没识别出来的项 → 可以用资源包认下来，并标注「中（来自资源包 X）」；
* 内置已识别的项 → 资源包只补空着的说明字段，并额外提供一个中文显示名；
* `cleanable` 与 `risk` **永远沿用内置结论**，资源包无权改写。

这条是刻意的：资源包是可以被随便替换的外部数据，不应该有影响「删不删」的能力。

## 写自己的包

1. 复制 `packs/apps-common.json` 改成自己的内容；
2. 界面右侧「资源包」面板里点「导入本地资源包」选这个文件，
   或在 `config/packs-user/` 下直接放一个 `.json`；
3. 导入时会做结构校验，不合法会直接告诉你哪一条有问题；
4. 导入后立刻生效，不需要重启服务。

自己写的包不会进版本库（`config/packs-user/` 已被 `.gitignore` 忽略）。
想分享给别人，把文件发过去让对方导入即可。

## 从 GitHub 更新

`packs/index.json` 是本目录的清单：

```json
{ "version": 1, "updated": "2026-10-12",
  "packs": ["windows-system.json", "apps-common.json",
            "games-platform.json", "games-titles.json"] }
```

界面上的「从 GitHub 检查更新」会按这个清单逐个下载并导入。
默认远端是**独立的公开仓库 [`O-Cleaner-packs`](https://github.com/2847439483-beep/O-Cleaner-packs)**
（与本目录保持同内容），任何人都可以更新；远端地址也可在界面上改成自己的。

> 历史说明：主仓库是私有的，raw 链接对外不可访问，2026-10-12 起改为
> 单独公开仓分发。本目录与公开仓内容一致，更新包时两边同步提交 ——
> 同步用 `python tools/sync_packs.py check|push`（check 只读比对，push 走
> GitHub Contents API），改完包跑一次 check 确认两边一致。

## 释义的三层优先级（2026-10-13 起）

同一个目录的说明文字可能来自三处，按以下优先级取第一层命中：

1. **具体软件条目**（apps-common / windows-system / games-*）：匹配位置最深，
   如 `pip\cache`、`wechat files` —— 永远优先；
2. **释义增强包（explanations.json）**：与内置通用规则（缓存、日志、下载目录、
   安装包……）匹配同一批路径，提供更详尽的「是什么 / 删了会怎样 / 何时该留」，
   让通用释义可以随「检查更新」远程改进而不用发新版；
3. **内置知识库文本**（config/filetypes.json）：保底。

三层都改不动「能不能删」（cleanable）与「风险」（risk）——结论永远由内置决定。
