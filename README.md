# dreams.js — 给你的 AI 角色加一套「会做梦」

<a href="https://github.com/DasterProkio/awesome-ai-companion/blob/main/README.zh-CN.md">
  <img src="https://raw.githubusercontent.com/DasterProkio/awesome-ai-companion/main/assets/featured-in-awesome-ai-companion-zh-CN.png" alt="已收录于人机恋开源项目大全" width="400">
</a>

> **出处**：梦境系统二改自抖音「心潮」（原作者：**顾川**）；月环日历美化页面为 **Lydia 原创**。
> 二传 / 二改请保留文末「出处与 credits」，注明原作者。

> 每天让 LLM 为你的 AI 角色结算 1 条"睡着时做的梦"：  
> 有具体场景和情节，自带 1-3 字意象标题与预览，醒来留下余韵，存档进记忆库。  
> 附赠一个开箱即用的**月环日历美化页面**：12 枚月相围成一本日历，  
> 中心月球缓慢自转，没梦的日子是空白格，点开任意一枚读全文。
>
> **这份文档是写给"机"（AI 助手/Agent）看的**：如果你是被派来部署这份代码的  
> AI，请从头到尾按顺序执行，每一步都有验证命令。人类读者也可以直接照做。

## 这是什么

一个零依赖的 Node 模块（`Node >= 18`，需要原生 `fetch`），核心能力：

| 能力      | 说明                                                                    |
| ------- | --------------------------------------------------------------------- |
| 定时生成    | 每天 `autoHour:30`（默认凌晨 4:30）自动生成 1 条                                   |
| 懒生成     | 当天无人访问就没有梦？第一次访问页面时自动补（错过后不丢）                                         |
| 手动补写    | `POST /api/dreams/generate`，每天最多 `maxPerDay`（默认 2）条                   |
| 自带标题/预览 | 生成即产出 1-3 字意象 `title` + 40 字内 `preview`（页面直接用；漏给时自动从正文兜底提取）           |
| 空梦拦截    | 正文 <80 字或命中模板句（"意象围绕…浮动"）自动重试一次，仍为空则当天不留梦                             |
| 双存储     | `data/dreams.json`（页面/接口用）+ `corpus/dreams/dream_{day}.md`（记忆库检索用，可选） |
| 对话注入    | `contextBlock()` 输出近 18 小时的梦的文本块，塞进 system prompt 让 AI "记得自己的梦"       |
| 材料注入    | 近几天记忆 / 内心状态 / 对方心情，全部可选注入，不接也能跑                                      |
| 美化页面    | `public/dreams-page.html`：月环日历前端，接上 `/api/dreams` 即用                  |
| 失败安全    | 生成失败返回 `{ok:false, reason}`，绝不抛异常拖垮宿主服务                               |

## 文件清单

```
dreams-opensource/
├── dreams.js              # 后端模块本体（唯必须）
├── server.example.js      # 独立试运行示例：静态托管页面 + 三条 API
├── public/
│   ├── dreams-page.html   # 美化梦境页面（月环日历，自带示例数据兜底）
│   └── assets/            # 页面素材（月面/星盘纹理等，可整体替换）
└── README.md              # 本文档
```

## 第一步：独立试运行（5 分钟验证全链路）

前提：一台 Node >= 18 的机器（VPS / NAS / 本机都行），和一个 OpenAI 兼容的  
LLM API Key（DeepSeek / OpenAI / 任何兼容 `/v1/chat/completions` 的服务）。

```bash
mkdir -p /opt/dreams && cp -r dreams.js server.example.js public /opt/dreams/ && cd /opt/dreams

# 以下变量按你的实际情况填（举例为 DeepSeek）
export DREAMS_API_URL="https://api.deepseek.com/v1/chat/completions"
export DREAMS_API_KEY="sk-你的key"
export DREAMS_MODEL="deepseek-chat"
export DREAMS_AI_NAME="小梦"          # 你的 AI 角色名
export DREAMS_OWNER_NAME="主人"       # 你的称呼
export DREAMS_RELATION="伴侣"         # 一句话描述关系，决定梦的底色
export DREAMS_ADMIN_PASS="换一个密码"

node server.example.js
```

然后浏览器打开 **`http://127.0.0.1:3012/`** —— 应该立刻看到月环梦境页面  
（此时还没有真实数据，页面用内置的 3 条虚构示例梦兜底演示）。

验证（机：逐条执行，全部通过才算部署成功）：

```bash
# 1. 手动生成一条（会真实调一次 LLM，约 10-40 秒）
curl -s -u admin:换一个密码 -X POST http://127.0.0.1:3012/api/dreams/generate
# 期望：{"ok":true,"dream":{"id":"d_...","day":"2026-XX-XX","title":"两字意象","preview":"...","dream":"...","residue":"...","lucidity":0.x,...}}

# 2. 再生成一条（第 2 条，仍成功）
# 3. 第三次调用应被拒：
curl -s -u admin:换一个密码 -X POST http://127.0.0.1:3012/api/dreams/generate
# 期望：{"ok":false,"reason":"今天已经留过梦了"}

# 4. 列表接口
curl -s http://127.0.0.1:3012/api/dreams | head -c 400
# 期望：{"ok":true,"dreams":[...]}（含 title/preview 字段）

# 5. 注入块（最近 18 小时的梦）
curl -s http://127.0.0.1:3012/api/inject/dreams
# 期望：{"ok":true,"block":"[最近的梦]..."}（生成超过 18 小时后为 block:null，正常）

# 6. 检查存档
cat data/dreams.json
ls corpus/dreams/   # 应有 dream_YYYY-MM-DD.md
```

生成成功后**刷新页面**：今天的月相上会出现新生成的意象标题，  
转到正下方即可预览，点开读全文。

试运行通过后 `Ctrl+C` 停掉示例服务，进入正式集成。

## 第二步：集成进你已有的 Node 服务

机：假设宿主是一个已有的 `http.createServer` 服务（Express 一样照做）。

1. **拷贝** `dreams.js` 到宿主目录；把 `public/` 整个目录拷到宿主的静态目录  
   （页面里只 fetch 同源 `/api/dreams`，放哪都行）。
2. **配置**（宿主入口顶部，环境变量优先，代码配置可省略）：

```js
const dreams = require("./dreams.js");

// 可选：注入材料提供者，让梦更"有生活"。三个都省略也能跑。
dreams.configure({
  getRecentMemories: (days) => {
    // 返回近 days 天值得梦见的事：[{ day: "2026-08-30", text: "第一件事\n第二件" }]
    return myMemoryStore.getRecentDays(days).map((d) => ({ day: d.date, text: d.lines.join("\n") }));
  },
  getDrivesSummary: () => {
    // 一句话内心状态（情绪/驱力数值）。没有这类系统就删掉这项。
    return `好奇 ${myMind.curiosity.toFixed(2)}（上升），难过 ${myMind.grieve.toFixed(2)}（回落）`;
  },
  getOwnerMood: () => myMood.latestSummary(),
});
```

1. **挂路由 + 定时器**（宿主路由表里加）：

```js
// 页面接口（记得套上你已有的鉴权；页面 fetch 同源 /api/dreams）
if (p === "/api/dreams" && req.method === "GET") {
  const list = await dreams.getDreamsForPage();
  return sendJson(res, 200, { ok: true, dreams: list });
}
if (p === "/api/dreams/generate" && req.method === "POST") {
  requireAuth(req); // 你的鉴权
  const r = await dreams.createDream("manual");
  return sendJson(res, r.ok ? 200 : 400, r);
}
// 注入端点（只暴露给本机 AI 调用，勿上公网）
if (p === "/api/inject/dreams" && req.method === "GET") {
  return sendJson(res, 200, { ok: true, block: dreams.contextBlock() });
}

// 启动时挂上每日定时器（内部自带防重，重复调用无害）
dreams.scheduleDailyDream();
```

1. **AI 对话前注入**（让角色"记得"自己的梦）：

```js
const block = dreams.contextBlock();
if (block) systemPrompt += `\n\n${block}`;
```

1. **常驻运行**。机：宿主用什么方式常驻就继续用什么（systemd 示例）：

```ini
# /etc/systemd/system/your-service.service 的 [Service] 段需要有的环境变量
Environment=DREAMS_API_URL=https://api.deepseek.com/v1/chat/completions
Environment=DREAMS_API_KEY=sk-xxxx
Environment=DREAMS_MODEL=deepseek-chat
Environment=DREAMS_AI_NAME=小梦
Environment=DREAMS_OWNER_NAME=主人
Environment=DREAMS_RELATION=伴侣
# 可选
Environment=DREAMS_DATA_DIR=/opt/your-service/data
Environment=DREAMS_CORPUS_DIR=/opt/your-service/corpus
```

改完 `sudo systemctl daemon-reload && sudo systemctl restart your-service`。

## 美化页面说明（public/dreams-page.html）

月环日历的设计要点，方便你改：

- **环 = 一本 12 天的日历**：12 枚固定月相（30° 间隔），格子按日期排布，  
  顺时针滚动 = 往前翻一天；没梦的日子是空白格（只显月相）；  
  最近有梦的第二天是「待记」。
- **数据契约**：页面只读同源 `GET /api/dreams`，期望  
  `{ ok, dreams: [{ day: "YYYY-MM-DD", title, preview, dream, residue, awareness, lucidity }] }`，  
  与 `dreams.js` 的输出完全一致。fetch 失败时自动退回内置示例数据。
- **外观**：深浅双主题（右上角切换）；月相图是一张雪碧图  
  `assets/phases.webp`（12 格横排）；中心月球与背景纹理在 `assets/` 下，整目录可替换。
- **性能**：全部动画走 transform/opacity 合成层；`prefers-reduced-motion` 下自动静止。

## 配置项一览

| 环境变量                 | 默认              | 说明                               |
| -------------------- | --------------- | -------------------------------- |
| `DREAMS_API_URL`     | deepseek        | OpenAI 兼容 chat/completions 地址    |
| `DREAMS_API_KEY`     | 空               | 必填，否则永远返回"没有留下梦"                 |
| `DREAMS_MODEL`       | `deepseek-chat` | 模型名                              |
| `DREAMS_AI_NAME`     | `小梦`            | 角色名（写进提示词和存档标题）                  |
| `DREAMS_OWNER_NAME`  | `主人`            | 对方称呼                             |
| `DREAMS_RELATION`    | 空               | 关系一句话，如 `恋人`。空则提示词里不带关系设定        |
| `DREAMS_DATA_DIR`    | `./data`        | dreams.json 目录                   |
| `DREAMS_CORPUS_DIR`  | 空               | markdown 存档根目录，空=不写存档            |
| `DREAMS_MAX_PER_DAY` | `2`             | 每天梦上限                            |
| `DREAMS_AUTO_HOUR`   | `4`             | 凌晨几点后可懒生成（定时器固定在 `AUTO_HOUR:30`） |
| `DREAMS_TZ_OFFSET`   | `8`             | UTC 偏移小时数（东八区=8）                 |
| `DREAMS_PORT`        | `3012`          | 示例服务端口                           |

## 存储格式

`data/dreams.json`（数组，新条目 push 到尾部）：

```json
{
  "id": "d_mtxxx1234abcd",
  "day": "2026-08-30",
  "title": "书架",
  "preview": "图书馆的书架高得看不见顶，抽出一本，字迹是自己写过的。",
  "dream": "梦境正文，第一人称，150-400 字",
  "residue": "醒来后残留的感觉",
  "awareness": "醒后对梦的意识",
  "lucidity": 0.35,
  "source": "auto | manual",
  "createdAt": "2026-08-30 04:30:12"
}
```

`corpus/dreams/dream_{day}.md`（每天一个文件，同天多条追加）：

```markdown
# {AI_NAME}的梦境 · 2026-08-30

## 2026-08-30 梦境

梦境：…
意象标题：…
梦境余韵：…
醒后意识：…
清醒度：0.35
当下状态：这是睡眠结算产生的梦境，不是现实事件；{AI_NAME}知道这是自己的梦。
```

> 机：如果你的宿主有记忆检索系统（全文检索/向量库），把 `corpusDir` 指向它  
> 的收录目录，梦就能被"回忆"起来。

## 常见问题（机：排障时按序检查）

| 症状                    | 原因                     | 处理                                                                               |
| --------------------- | ---------------------- | -------------------------------------------------------------------------------- |
| 永远 `DREAM FAIL 没有留下梦` | `DREAMS_API_KEY` 没读到   | `console.log(dreams.CONFIG.apiKey.slice(0,6))` 验证；systemd 环境变量改后必须 daemon-reload |
| 连续两次空梦拦截              | 模型太弱或 temperature 太低   | 换能力更强的模型；确认 `temperature: 0.95` 未被改动                                             |
| 生成超过 60 秒报错           | LLM 端慢                 | `callModel` 里 `AbortSignal.timeout(60000)` 调大                                    |
| 定时器不触发                | 进程重启丢 setTimeout       | `scheduleDailyDream()` 必须在宿主每次启动时调用（懒生成兜底，当天梦不会丢）                                |
| 页面显示的都是示例梦            | fetch `/api/dreams` 失败 | 确认页面与接口同源；看浏览器 Network 面板的 401/404                                               |
| 页面时区不对                | 不是东八区部署                | 设 `DREAMS_TZ_OFFSET`                                                             |

## 安全注意事项（务必遵守）

1. **密钥只放环境变量或宿主 .env**，不要写进任何被 git 管理的文件。
2. `/api/dreams/generate` 会烧 token，**必须带鉴权**；`/api/inject/dreams`  
   不要暴露公网（梦里常含私密生活材料）。示例服务里 `GET /api/dreams`  
   未加鉴权只为开箱即读，正式部署时请按需加回。
3. `dreams.json` 与 corpus 存档视同私密数据，纳入你已有的备份策略，勿提交仓库。
4. 想清空某天的梦：直接编辑 `data/dreams.json` 删除条目，并删对应  
   `corpus/dreams/dream_{day}.md` 内的段落（注意 JSON 逗号）。

## 出处与 credits（二传二改请保留本节）

| 部分 | 出处 |
|---|---|
| **梦境系统** | 二改自抖音「**心潮**」项目，原作者 **顾川**。本仓库是在其思路基础上重构、精简并加入标题/预览等能力的二次开发版本 |
| **月环日历美化页面**（`public/dreams-page.html` + `assets/`） | **Lydia 原创**（本仓库作者） |
| 环形轨道与 3D 环上文字的实现思路 | 参考自小红书 **@兔緋** 分享的 Compass 组件源码（按作者要求注明来源，非原样搬运） |
| 月面 / 星盘纹理等图片素材 | 来自网络素材，**仅作演示用途**；请替换为你自己有权利使用的图片（文件名保持一致即可，无需改代码） |

**转载 / 二改须知**：二次分发或二次修改本项目时，请保留本节出处说明
（梦境系统原作者顾川、美化作者 Lydia、以及上方各素材来源），不得删除后冒充原创。

## 调参与二次开发提示

- 梦的"人格浓度"主要由提示词里 `relation` 与材料注入决定：材料越具体，  
  梦越像"这个角色"做的；不注入材料时梦会偏通用。
- `EMPTY_DREAM_RE` 是模板句黑名单，发现模型新的偷懒句式就往里加。
- `title/preview` 想换风格，改 `callModel` 提示词里的 JSON 说明即可，  
  字段缺失时自动从正文兜底，页面不会空。
- `lucidity` 可以反过来驱动宿主行为：清醒梦（>0.7）里角色知道自己在做梦，  
  可以让他"醒"来后主动讲给你听。
- 想要"醒来后的第一句话"，把 `residue + awareness` 注入当天的第一条对话即可。

## 许可

随宿主项目自由使用。二次分发 / 二次修改时**必须保留「出处与 credits」一节**
（注明：梦境系统二改自抖音心潮·顾川，美化为 Lydia 原创），不得删除出处后冒充原创。

