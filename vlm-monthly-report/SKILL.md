---
name: vlm-monthly-report
description: 从量子位 WP API 实拉 30 天数据，制作多模态大模型产业月报 HTML（五大主线+Top10 实图+分类流+产业地图+认知升级），含完整设计系统与校验清单；代理无关，适用 Claude Code / Codex 等任何能跑 shell 的 agent
---

> 公开版说明：历史实例、模型名、日期和跑分仅作写作与版式示例；发布前应按原文复核。只嵌入可公开使用的图片，未确认授权时改用自制示意或仅保留来源链接。

# VLM 产业月报制作技能（vlm-monthly-report）

> 适用任何能执行 shell 与读写文件的 coding agent（Claude Code / Codex 等）。自包含文档：数据命令、脚本、设计系统、编辑规则、校验清单全部在内，无需其他依赖。
>
> 参考成品：仓库中的 sample/index.html（虚构条目和原创插图，仅演示版式）。

## 0. 何时使用

用户说「**帮我调研多模态大模型最近 30 天 / 一个月的进展，整理成 HTML**」之类需求时使用本技能。

- 输出文件：`outputs/vlm-monthly-brief-YYYY-MM-DD.html`（日期 = 出刊日）
- 报告标题：`AI 产业动态月报 | 多模态大模型 VLM/MLLM | 30 天特辑 YYYY-MM-DD ~ YYYY-MM-DD`
- 一个约 1MB 的自包含 HTML（配图 base64 内嵌，离线可看，无外部依赖）

## 1. 流程总览

```
① 定窗口 → ② WP API 拉数 → ③ 压 digest → ④ 筛选分类 + 编辑
→ ⑤ 拉原文配图 → ⑥ 生成 HTML（含 CSS/JS 模板）→ ⑦ 校验 → 出刊
```

**铁律：数据一律实拉，不做任何「网络搜索 + 模型记忆」合成。** 搜索工具（任何 agent 的）都可能返回编造内容——本文所有事实必须来自 WP API 返回的 `content`。

## 2. 第①步：窗口衔接

1. 列出 `outputs/` 下已有 `vlm-monthly-brief-*.html`，读最近一期 `<title>` 的窗口终点。
2. 本期窗口 = 出刊日往前 30 天，但**起点不得早于上期终点 + 1 天**（不重复收录；上期 09-21 止 → 本期起点取 09-28）。
3. 窗口写进标题与 nav，两处一致。

## 3. 第②步：量子位 WP API 拉数

量子位是 WordPress 站，REST API 公开。Git Bash `curl` 或任何 HTTP 客户端均可（带 UA）：

```bash
mkdir -p work/vlm_30d && cd work/vlm_30d
START="2026-09-28T00:00:00"   # 窗口起点
for p in 1 2 3; do
  curl -s --max-time 30 \
    "https://www.qbitai.com/wp-json/wp/v2/posts?per_page=100&page=$p&after=$START&_fields=id,date,link,title,content" \
    -H "User-Agent: Mozilla/5.0" -o "posts_p$p.json" -w "page$p: %{http_code} %{size_download}B\n"
  sleep 2
done
```

30 天约 200–260 篇，3 页拉完（第 3 页接近空即到头）。失败重试即可。

## 4. 第③步：压缩 digest

原始 JSON 很大（单页可达 19MB），先转成可读摘要文件再分析：

```bash
python - <<'EOF'
import json, re, html
posts = []
for p in [1,2,3]:
    posts += json.load(open(f'posts_p{p}.json', encoding='utf-8'))
def strip(h):
    t = re.sub(r'<[^>]+>', ' ', h or '')
    t = html.unescape(t)
    return re.sub(r'\s+', ' ', t).strip()
lines = []
for po in posts:
    lines.append(f"{po['date'][:10]} | {po['link']} | {strip(po['title']['rendered'])}\n    {strip(po['content']['rendered'])[:400]}")
open('digest.txt', 'w', encoding='utf-8').write('\n'.join(lines))
print('total:', len(posts), posts[-1]['date'][:10], '->', posts[0]['date'][:10])
EOF
```

通读 digest（200 多篇、约 100KB，分段读全），按第 5 节标准筛选。

## 5. 第④步：筛选与编辑规则

### 收录标准（多模态相关）

收录：基座/多模态模型发布与跑分、生图/视频/3D/世界模型、具身 VLM 与世界动作模型、OCR/文档解析、语音交互、端侧多模态、RSI 与模型安全、Agent 工具（多模态相关优先）、算力/推理基建（token 经济学视角）、重大资本事件（IPO/并购/融资，须与多模态格局相关）。

排除：纯汽车产业新闻（车展/销量/电池）、纯消费电子发布会、纯 PR 稿（**收录时打 [PR] 角标**）、与 AI 无关的融资。

### 分类体系（5 类，贯穿筛选按钮与卡片 data-cat）

| data-cat | 名称 | 图标 | 含义 |
|---|---|---|---|
| model | 基础模型 | 🧠 | 基座/多模态/生成模型发布与评测 |
| agent | Agent 工具 | 🤖 | 智能体框架、Harness、办公 Agent |
| embodied | 具身智能 | 🦾 | 机器人、VLA、世界动作模型、物理 AI |
| biz | 商业应用 | 💼 | 终端合作、融资 IPO、商业落地 |
| eco | 生态基建 | ⚡ | 算力、推理优化、开源生态、安全 |

### 结构配额

- **五大主线**：从全部素材里提炼 5 条跨事件叙事线（如「半价旗舰价格战」「多模态生成专业化」「世界模型并购与开源」「具身大脑 VLM 化」「RSI 与安全」），每条 = 主线名 + 3–4 句叙事 + 6 个左右事件链（带日期）+ 一句「判断」
- **Top 10**：按「对多模态格局的影响权重」排序，每卡写 `为什么重要`
- **分类资讯流**：约 46 条（13/9/9/8/7 上下），每卡 1–2 句摘要
- **科学前沿特辑**：4 条 AI for Science
- **认知升级**：1 条核心判断（金卡）+ 2 条次级判断 + 下期观察 5 条

### 编辑规则（用户明确反馈过的坑）

1. **每张卡片的 dateline 必须带原文跳转链接**：`<a class="lnk" href="WP API 的 link 字段" target="_blank">原文 ↗</a>`。首例漏了全部链接被用户指出。
2. 金标 ⭐ 给「多模态能力边界关键动态」（约 8–10 处）；金卡 `.news-card.gold` 是 `border-left:3px solid var(--gold)`。
3. 判断/认知升级**必须锚定当期具体事件与数字**，不写空话。
4. 数字一律用文章原文的数字（跑分、价格、融资额），不估算。
5. nav 的「覆盖 N 条」与分类小计、实际卡片数一致（校验脚本会数）。
6. **技术断言必须区分「原文事实」与「推断」**（2026-11-06 教训）：模型架构/实现机理类信息官方经常不公开——实例：GPT-Image 2 研究负责人陈博远被直接问到时**拒绝回答「扩散还是自回归」**，只肯说「通用模型」「图像领域的 GPT」。禁止把社区猜测写成「普遍认为」；推断要么标注为推断，要么不写。核验信源现实：Bing 搜索全是 SEO 垃圾站、openai.com 对 curl 返回 403、DuckDuckGo 连不上——**本机唯一可靠核验信源是量子位 WP API 全文搜**（`/wp-json/wp/v2/posts?search=关键词&_fields=id,date,link,title,content`，命中后拉 content 提取原句；2026-11-06 用它当场翻案「GPT-Image 系普遍被认为是 AR 骨干」这句未证实断言，并顺藤找到负责人拒答的原文）。

## 6. 第⑤步：原文配图管线

Top 10 封面（10 张）+ 五大主线横幅（5 张）配**原文实图**；资讯流缩略图用占位。

### 环境注意

- **默认 python（miniconda）没有 PIL**；用 `py -3.12`（有 PIL 13 + requests）。任何图像处理一律 `py -3.12`。
- 图床 `i.qbitai.com` 可直连，**必须带 `Referer: https://www.qbitai.com/`**，否则 403。

### 提取 → 下载 → base64

```bash
py -3.12 - <<'EOF'
import json, re, base64, io, requests
from PIL import Image
posts = {}
for p in [1,2,3]:
    for po in json.load(open(f'posts_p{p}.json', encoding='utf-8')):
        posts[po['id']] = po
targets = {'top1': <文章id>, 'top2': <id>, ... 'ml1': <id>, ...}  # 按卡片规划填
data = {}
for name, pid in targets.items():
    imgs = re.findall(r'<img[^>]+src="([^"]+)"', posts[pid]['content']['rendered'])
    for url in imgs[:5]:                       # 逐张试，找到能用的为止
        if 'qbitai.com' not in url: continue   # 外站 CDN 基本会挂
        try:
            r = requests.get(url, timeout=25, headers={'User-Agent':'Mozilla/5.0','Referer':'https://www.qbitai.com/'})
            r.raise_for_status()
            im = Image.open(io.BytesIO(r.content)).convert('RGB')
            if im.width < 300: continue        # WeChat CDN 常给 140px 小图
            w = 900 if name.startswith('ml') else 640
            if im.width > w: im = im.resize((w, int(im.height*w/im.width)), Image.Resampling.LANCZOS)
            buf = io.BytesIO(); im.save(buf, 'JPEG', quality=76)   # webp/gif 一律转 JPEG
            data[name] = 'data:image/jpeg;base64,' + base64.standard_b64encode(buf.getvalue()).decode()
            break
        except Exception: continue
json.dump(data, open('img_data.json','w'))
print('ok:', len(data), '/', len(targets))
EOF
```

失败替换策略（按优先级）：同文章第 2–5 张图 → 关联文章（同主题另一篇）首图 → 设计化占位（noimg div）。已知不可用源：`mmbiz.qpic.cn`（微信 CDN，多为 140px 小图）、`toutiaoimg.com`（403）。gif 取 PIL 首帧。

### 注入（关键技巧：标记占位法）

生成 HTML 时**直接在卡片里写标记注释**（主线用整标签占位，封面用内嵌标记）：

```html
<div class="cover"><!--IMG:top1--></div>
<!--IMGTAG:ml1-->
```

配图就绪后一轮替换，**不要做按位置插入**（第 2 期曾因位置注入脚本漏 body 开标签返工）：

```bash
py -3.12 - <<'EOF'
import json, re
path = r'outputs/vlm-monthly-brief-YYYY-MM-DD.html'
raw = open(path, encoding='utf-8').read()
data = json.load(open('img_data.json'))
for k, uri in data.items():
    raw = raw.replace(f'<!--IMG:{k}-->', f'<img src="{uri}" alt="Top 配图">')
    raw = raw.replace(f'<!--IMGTAG:{k}-->', f'<img class="mlimg" src="{uri}" alt="主线配图">')
# 未命中的标记：封面换占位、主线整行删除
raw = re.sub(r'<div class="cover"><!--IMG:[\w-]+--></div>',
             '<div class="cover"><div class="noimg"><span class="big">✨</span>原文配图暂缺</div></div>', raw)
raw = re.sub(r'[ \t]*<!--IMGTAG:[\w-]+-->\n?', '', raw)
open(path, 'w', encoding='utf-8').write(raw)
assert '<!--IMG' not in raw and '<!--IMGTAG' not in raw
print('注入完成')
EOF
```

## 7. 第⑥步：HTML 结构与设计系统

### 章节树（月报全量结构）

```
<nav>              品牌条 + 窗口 + 「覆盖 N 条 · 源自 M 篇」
<div class=filters> 6 个按钮: all/model/agent/embodied/biz/eco
<div class=container>
  <section id=mainlines>  h2 + 5 × .mainline（h3 主线名+no 徽标 / p 叙事 / img.mlimg / .chain 事件链 / .verdict 判断）
  <section id=top10>      h2 + .hero-grid（首卡 grid-column:1/-1；10 × .fcard: .cover / .body(rank,h3,dateline,summary,why)）
  <section id=stream>     h2 + 5 × .cat-block（.cat-head + N × .news-card: .thumb/.nc-body(h4,dateline,summary)）
  <section id=sci>        h2 + .sci-grid（4 × .sci-card: h4,.d,p）
  <section id=map>        h2 + .map-grid（4 × .map-card: h4 + ul/li）
  <section id=cognition>  h2 + .keymain + .subgrid(2×.subfind) + .nextwatch
<footer>           数据源 + 窗口 + 期号
<script>           分类过滤 JS（见下）
```

### 卡片字段规格

- `.mainline .ev`：`<b>主体</b> 一句话<small>MM-DD</small>`，事件间 `.arr` 箭头
- `.fcard`：`<span class="rank">TOP N</span>` + h3 标题 + `.dateline`（日期 ｜ 主体 · 原文链接）+ `.summary` + `.why`（为什么重要）
- `.news-card`：h4 标题（金标动态加 `class="gold"` 与 `<span class="star">⭐</span>`）+ dateline + summary；缩略图一律 noimg 占位
- 所有分类元素带 `data-cat="model|agent|embodied|biz|eco"`（含 sci/map 之外的卡片，参与过滤）

### 骨架样例（可直接复制改内容；文案风格即规格）

```html
<nav>
  <div class="brand">🛰️ AI 产业动态月报 · <span class="accent">多模态大模型 VLM/MLLM</span></div>
  <div class="meta">30 天特辑 <b>2026-09-28 ~ 2026-10-28</b> ｜ 覆盖 <b>56</b> 条精选动态 · 源自 207 篇报道</div>
</nav>

<!-- 五大主线卡（配图用 IMGTAG 标记占位） -->
<div class="mainline">
  <h3><span class="no">主线 1</span>旗舰季对轰:「半价旗舰」成主旋律</h3>
  <p>3–4 句叙事：谁开场、30 天里发生了什么、共同点是什么。数字与日期来自原文。</p>
  <!--IMGTAG:ml1-->
  <div class="chain">
    <span class="ev"><b>DevDay</b> GPT-6.1 Sol+Dots<small>09-30</small></span><span class="arr">→</span>
    <span class="ev"><b>Gemini 4 Argon</b> 空降多榜第一<small>10-01</small></span><span class="arr">→</span>
    <span class="ev"><b>Haiku 5.5</b> 斩杀线价<small>10-08</small></span>
  </div>
  <div class="verdict"><b>判断:</b>一句话结论——本质是什么、谁受益、下一步看什么。</div>
</div>

<!-- Top10 卡（首卡 grid 自动通栏；封面用 IMG 标记占位） -->
<div class="fcard" data-cat="model">
  <div class="cover"><!--IMG:top1--></div>
  <div class="body">
    <span class="rank">TOP 1</span>
    <h3>Gemini 4 Argon 正式发布:谷歌重回御三家,RSI 加持 + API 限时半价</h3>
    <div class="dateline">10-01 空降多榜第一 ｜ 10-16 全球开放 ｜ <b>Google DeepMind</b> · <a class="lnk" href="https://www.qbitai.com/2026/10/503701.html" target="_blank">原文 ↗</a></div>
    <div class="summary">2–3 句:主打什么能力、关键数字(价格/跑分/上下文)、评测结论。数字抄原文。</div>
    <div class="why"><b>为什么重要:</b>1–2 句:这件事改变了什么格局/确立了什么趋势。</div>
  </div>
</div>

<!-- 资讯流卡（金标版；普通卡去掉 class="gold" 与 star） -->
<div class="news-card gold" data-cat="model">
  <div class="thumb"><div class="noimg"><span class="big">⚡</span>原文配图</div></div>
  <div class="nc-body">
    <h4>GPT-6.1 Terra 首批评测:1 折价格获得 Astra 级电脑操作<span class="star">⭐</span></h4>
    <div class="dateline">10-27 ｜ OpenAI · <a class="lnk" href="https://www.qbitai.com/2026/10/505719.html" target="_blank">原文 ↗</a></div>
    <div class="summary">1–2 句摘要,带 1–2 个关键数字,结尾可加一句对比或定位。</div>
  </div>
</div>

<!-- 科学前沿卡 -->
<div class="sci-card" data-cat="model">
  <h4>陶哲轩:数学 2.0 时代已至</h4>
  <div class="d">10-11 ｜ SAIR 演讲后正式发文</div>
  <p>2–3 句:事件 + 关判断 + 一句引语或数字。</p>
</div>
```

标题文案规格：**「主体 + 动作:副题（数字/对比/反差）」**冒号结构，副题里放最有信息量的数字或冲突（例：「Claude Haiku 5.5 发布:跑分暴击 GPT-6 Luna,价格斩国产 Flash」）。

### 分类过滤 JS（页尾）

```html
<script>
(function(){
  var btns = document.querySelectorAll('.fbtn');
  var cards = document.querySelectorAll('[data-cat]');
  btns.forEach(function(b){
    b.addEventListener('click', function(){
      btns.forEach(function(x){ x.classList.remove('active'); });
      b.classList.add('active');
      var f = b.getAttribute('data-filter');
      cards.forEach(function(c){
        var cat = c.getAttribute('data-cat');
        var isBlock = c.classList.contains('cat-block');
        if (f === 'all' || cat === f) { c.style.display = isBlock ? '' : 'flex'; }
        else { c.style.display = 'none'; }
      });
    });
  });
})();
</script>
```

### 设计系统（完整 CSS，直接内嵌 <style>）

深色 slate 产业简报主题；换主题只动 `:root` tokens。

```css
:root{
  --bg:#0f172a; --card:#1e293b; --card2:#233046;
  --blue:#38bdf8; --purple:#a78bfa; --green:#34d399; --amber:#fbbf24; --red:#f87171;
  --t1:#f1f5f9; --t2:#cbd5e1; --t3:#94a3b8; --gold:#fbbf24;
}
*{margin:0;padding:0;box-sizing:border-box}
body{background:var(--bg);color:var(--t2);font-family:-apple-system,BlinkMacSystemFont,"Segoe UI","PingFang SC","Hiragino Sans GB","Microsoft YaHei",sans-serif;line-height:1.65}
a{color:var(--blue);text-decoration:none}
a.lnk{font-size:12px;color:var(--blue)}
a.lnk:hover{text-decoration:underline}
img{max-width:100%;display:block}
nav{position:sticky;top:0;z-index:100;background:rgba(15,23,42,.88);backdrop-filter:blur(14px);-webkit-backdrop-filter:blur(14px);border-bottom:1px solid rgba(148,163,184,.15);padding:14px 24px;display:flex;flex-wrap:wrap;align-items:center;gap:14px;justify-content:space-between}
nav .brand{font-size:17px;font-weight:700;color:var(--t1)}
nav .brand .accent{color:var(--blue)}
nav .meta{font-size:13px;color:var(--t3)}
nav .meta b{color:var(--amber);font-weight:600}
.filters{display:flex;gap:8px;flex-wrap:wrap;padding:14px 24px 0;max-width:1320px;margin:0 auto}
.fbtn{border:1px solid rgba(148,163,184,.25);background:transparent;color:var(--t3);padding:6px 16px;border-radius:999px;font-size:13px;cursor:pointer;transition:.2s}
.fbtn:hover{border-color:var(--blue);color:var(--t2)}
.fbtn.active{background:var(--blue);border-color:var(--blue);color:#0f172a;font-weight:600}
.container{max-width:1320px;margin:0 auto;padding:28px 24px 60px}
section{margin-bottom:44px}
h2.sec{font-size:21px;color:var(--t1);font-weight:700;margin-bottom:6px;display:flex;align-items:center;gap:10px}
h2.sec .bar{width:5px;height:22px;border-radius:3px;background:var(--blue);flex:0 0 5px}
.sec-sub{font-size:13px;color:var(--t3);margin-bottom:18px}
.mainline{background:linear-gradient(135deg,#1a2540,#1e293b);border:1px solid rgba(56,189,248,.2);border-radius:16px;padding:24px 26px;margin-bottom:20px}
.mainline h3{color:var(--t1);font-size:18px;margin-bottom:10px;display:flex;align-items:center;gap:10px;flex-wrap:wrap}
.mainline h3 .no{font-family:Georgia,serif;font-size:15px;color:var(--amber);border:1px solid rgba(251,191,36,.4);border-radius:8px;padding:1px 10px}
.mainline p{font-size:14px;margin-bottom:12px}
.mainline .mlimg{width:100%;aspect-ratio:16/6;object-fit:cover;border-radius:10px;border:1px solid rgba(148,163,184,.15);margin:12px 0 2px;display:block}
.mainline .chain{display:flex;flex-wrap:wrap;gap:6px;align-items:center;margin:10px 0 4px}
.mainline .ev{font-size:11.5px;background:rgba(15,23,42,.55);border:1px solid rgba(148,163,184,.18);border-radius:8px;padding:4px 10px;color:var(--t2);white-space:nowrap}
.mainline .ev b{color:var(--amber);font-weight:600}
.mainline .ev small{color:var(--t3);margin-left:5px}
.mainline .arr{color:var(--t3);font-size:11px}
.mainline .verdict{margin-top:12px;padding-top:12px;border-top:1px dashed rgba(148,163,184,.2);font-size:13px;color:var(--t2)}
.mainline .verdict b{color:var(--blue)}
.hero-grid{display:grid;grid-template-columns:repeat(2,1fr);gap:20px}
.hero-grid .fcard:first-child{grid-column:1/-1}
.fcard{background:var(--card);border:1px solid rgba(148,163,184,.14);border-radius:16px;overflow:hidden;transition:.25s;display:flex;flex-direction:column;box-shadow:0 4px 18px rgba(0,0,0,.25)}
.fcard:hover{transform:translateY(-4px);box-shadow:0 14px 34px rgba(0,0,0,.45);border-color:rgba(56,189,248,.4)}
.fcard .cover{position:relative;aspect-ratio:16/8;overflow:hidden;background:#16202f}
.fcard .cover .noimg{width:100%;height:100%;display:flex;align-items:center;justify-content:center;color:var(--t3);font-size:12px;text-align:center;flex-direction:column;gap:2px;background:linear-gradient(135deg,#16202f,#1c2a42)}
.fcard .cover .noimg .big{font-size:30px;opacity:.5}
.fcard .body{padding:18px 20px 20px;display:flex;flex-direction:column;gap:10px;flex:1}
.fcard .rank{display:inline-block;background:rgba(251,191,36,.95);color:#0f172a;font-weight:800;font-size:13px;padding:3px 12px;border-radius:999px;align-self:flex-start}
.fcard h3{font-size:17px;color:var(--t1);line-height:1.5}
.fcard .dateline{font-size:12px;color:var(--t3)}
.fcard .dateline b{color:var(--t2);font-weight:500}
.fcard .summary{font-size:14px}
.fcard .why{background:rgba(56,189,248,.09);border-left:3px solid var(--blue);border-radius:0 10px 10px 0;padding:10px 14px;font-size:13px;color:var(--t2)}
.fcard .why b{color:var(--blue)}
.tag{display:inline-block;font-size:11px;padding:2px 10px;border-radius:999px;font-weight:600;letter-spacing:.3px}
.tag.model{background:rgba(56,189,248,.15);color:var(--blue)}
.tag.agent{background:rgba(167,139,250,.15);color:var(--purple)}
.tag.embodied{background:rgba(52,211,153,.14);color:var(--green)}
.tag.biz{background:rgba(251,191,36,.14);color:var(--amber)}
.tag.eco{background:rgba(248,113,113,.14);color:var(--red)}
.cat-block{margin-top:8px}
.cat-head{display:flex;align-items:center;gap:10px;margin:34px 0 16px}
.cat-head .icon{font-size:22px}
.cat-head h3{font-size:18px;color:var(--t1);font-weight:700}
.cat-head .cnt{font-size:12px;color:var(--t3);background:var(--card);padding:2px 10px;border-radius:999px}
.cat-head .line{flex:1;height:1px;background:linear-gradient(90deg,rgba(148,163,184,.25),transparent)}
.news-card{background:var(--card);border:1px solid rgba(148,163,184,.12);border-radius:14px;padding:16px;display:flex;gap:16px;transition:.25s;margin-bottom:12px;position:relative;box-shadow:0 3px 12px rgba(0,0,0,.2)}
.news-card:hover{transform:translateY(-4px);box-shadow:0 12px 28px rgba(0,0,0,.4)}
.news-card.gold{border-left:3px solid var(--gold)}
.news-card .thumb{width:132px;min-width:132px;height:88px;border-radius:10px;overflow:hidden;background:#16202f;align-self:flex-start}
.news-card .thumb .noimg{width:100%;height:100%;display:flex;align-items:center;justify-content:center;color:var(--t3);font-size:11px;flex-direction:column;gap:2px;background:linear-gradient(135deg,#16202f,#1c2a42)}
.news-card .thumb .noimg .big{font-size:20px;opacity:.5}
.news-card .nc-body{flex:1;display:flex;flex-direction:column;gap:6px;min-width:0}
.news-card h4{font-size:15px;color:var(--t1);line-height:1.5}
.news-card .dateline{font-size:12px;color:var(--t3)}
.news-card .summary{font-size:13px}
.star{color:var(--gold);margin-left:4px}
.map-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:18px}
.map-card{background:var(--card);border:1px solid rgba(148,163,184,.12);border-radius:14px;padding:18px}
.map-card h4{font-size:15px;color:var(--t1);margin-bottom:12px}
.map-card ul{list-style:none}
.map-card li{font-size:13px;padding:6px 0;border-bottom:1px dashed rgba(148,163,184,.12);color:var(--t2)}
.map-card li:last-child{border-bottom:none}
.map-card li b{color:var(--t1);font-weight:600}
.keymain{background:linear-gradient(135deg,rgba(251,191,36,.12),rgba(30,41,59,.6));border:1.5px solid rgba(251,191,36,.45);border-radius:14px;padding:22px 24px;margin-bottom:18px}
.keymain .kmtag{display:inline-block;font-size:11.5px;font-weight:700;color:#0f172a;background:var(--gold);border-radius:999px;padding:2px 14px;margin-bottom:10px;letter-spacing:1px}
.keymain h3{color:var(--t1);font-size:19px;margin-bottom:10px;line-height:1.5}
.keymain p{font-size:14px;margin-bottom:0}
.keymain p b{color:var(--gold)}
.subgrid{display:grid;grid-template-columns:repeat(2,1fr);gap:14px;margin-bottom:18px}
.subfind{background:rgba(15,23,42,.5);border:1px solid rgba(148,163,184,.15);border-radius:12px;padding:14px 16px}
.subfind .sno{display:inline-block;font-size:10.5px;color:var(--t3);border:1px solid rgba(148,163,184,.3);border-radius:6px;padding:0 8px;margin-bottom:6px}
.subfind h4{font-size:14px;color:var(--t1);margin-bottom:6px}
.subfind p{font-size:12.5px;margin:0;color:var(--t2)}
.nextwatch{background:rgba(56,189,248,.07);border-left:3px solid var(--blue);border-radius:0 12px 12px 0;padding:14px 18px;font-size:13px}
.nextwatch b{color:var(--blue)}
.sci-grid{display:grid;grid-template-columns:repeat(2,1fr);gap:14px}
.sci-card{background:var(--card);border:1px solid rgba(148,163,184,.12);border-radius:14px;padding:16px 18px}
.sci-card h4{font-size:14.5px;color:var(--t1);margin-bottom:6px}
.sci-card .d{font-size:11.5px;color:var(--t3);margin-bottom:8px}
.sci-card p{font-size:13px}
footer{text-align:center;font-size:12px;color:var(--t3);padding:26px;border-top:1px solid rgba(148,163,184,.1);line-height:1.9}
footer b{color:var(--t2)}
@media(max-width:1000px){.hero-grid{grid-template-columns:1fr}.hero-grid .fcard:first-child{grid-column:auto}.map-grid{grid-template-columns:1fr}.sci-grid{grid-template-columns:1fr}}
@media(max-width:760px){.subgrid{grid-template-columns:1fr}.news-card .thumb{width:100%;min-width:100%;height:150px}}
@media(max-width:680px){.hero-grid{grid-template-columns:1fr}.sci-grid{grid-template-columns:1fr}.news-card{flex-direction:column}nav{padding:12px 16px}.filters{padding:10px 16px 0}.container{padding:18px 16px 50px}}
```

## 8. 第⑦步：出刊前校验（必须全绿）

```bash
python - <<'EOF'
from html.parser import HTMLParser
import re
VOID = {'meta','link','br','hr','img','input','source','path','rect','circle','line','use','marker','polygon','text'}
class P(HTMLParser):
    def __init__(self):
        super().__init__(convert_charrefs=True)
        self.stack=[]; self.errors=[]
    def handle_starttag(self,tag,attrs):
        if tag not in VOID: self.stack.append((tag,self.getpos()[0]))
    def handle_endtag(self,tag):
        if tag in VOID: return
        if not self.stack: self.errors.append(f'extra </{tag}> L{self.getpos()[0]}'); return
        top,ln=self.stack.pop()
        if top!=tag: self.errors.append(f'mismatch <{top}> L{ln} by </{tag}> L{self.getpos()[0]}')
raw = open(r'outputs/vlm-monthly-brief-YYYY-MM-DD.html', encoding='utf-8').read()
p = P(); p.feed(raw)
assert not p.stack and not p.errors, (p.stack[:5], p.errors[:5])          # ① 标签配平
fcards = re.findall(r'<div class="fcard"[\s\S]{0,150}', raw)
assert len(fcards) == 10 and all('<div class="cover">' in f for f in fcards)  # ② Top10 全带 cover
cats = re.findall(r'data-cat="(\w+)"', raw)
from collections import Counter
assert set(cats) <= {'model','agent','embodied','biz','eco'}              # ③ data-cat 合法
links = re.findall(r'href="https://www\.qbitai[^"]+"', raw)
assert len(links) >= 50                                                   # ④ 链接密度
assert raw.count('data:image/jpeg') >= 10                                 # ⑤ 配图就位
assert '<!--IMG:' not in raw                                              # ⑥ 标记全部替换
print('OK |', dict(Counter(cats)), '| links:', len(links), '| size:', len(raw)//1024, 'KB')
EOF
```

校验不过不出刊；HTML 标签配平是最常见故障（批量注入易破坏结构），修完必须重跑。

## 9. 已知坑（历史返工原因）

| 坑 | 对策 |
|---|---|
| 漏原文链接（首例被用户指出） | 写卡片时同步写链接，不后补 |
| 搜索工具返回编造内容（模型名/日期全是幻觉） | 事实一律来自 WP API 实拉 content；缩写模型名要原文确认真身 |
| WeChat CDN 小图 / toutiao 403 / GDI+ 不支持 webp | 换同文章第 2 张或关联文章图；PIL 统一转 JPEG |
| 默认 python 无 PIL | 图像处理走 `py -3.12` |
| 按位置注入 HTML 吞掉标签 | 用 `<!--IMG:key-->` 标记占位法，一轮 replace |
| Windows 控制台 GBK 乱码 | 所有 python 前加 `export PYTHONIOENCODING=utf-8`；别写含中文的 .ps1 |
| 写 .bat 用 LF 换行导致 cmd 卡死 | 本流程不写 .bat；如写必须转 CRLF |

## 10. 周报模式差异（同管线缩水版）

用户说「最近一周」时：文件名 `vlm-industry-brief-YYYY-MM-DD.html`，WP API `after=` 7 天前（约 60–80 篇单页拉完），Top10 缩为 Top5，无科学特辑，其余同月报。上期窗口衔接规则相同。**注意：上期是月报时，起点 = 月报终点次日**（月报终点 10-28 → 周报起点 10-29，别默认「今天减 7 天」把中间空窗的文章漏掉——2026-11-06 期曾因此漏掉 13 篇含 Gemini 4 Carbon 重磅，重拉返工）。

### 图文并茂（硬要求，2026-11-06 用户反馈后的规格）

周报与月报一律三层配图全部用**原文实图**，占位 div 必须清零：

| 层 | 规格 | 提取 |
|---|---|---|
| Top 封面 | 640px 宽, quality 76 | 文章第 1 张可用图 |
| 主线横幅 `.mlimg` | 900px 宽, quality 74（CSS `aspect-ratio:16/6;object-fit:cover`） | 同文章**第 2 张起**（第 1 张常已被封面/缩略图占用） |
| 资讯流缩略图 | 264px 宽（2x retina）, quality 70 | 文章第 1 张可用图，配 `.news-card .thumb img{width:100%;height:100%;object-fit:cover}` |

- 缩略图注入：每张卡的 noimg 占位 emoji 唯一，regex 整块替换 `<div class="thumb"><div class="noimg">…</div></div>` → `<div class="thumb"><img src=…></div>`；主线横幅按第 N 个 `<div class="chain">` 依次前插。
- 单图失败换同文章第 2–5 张 → 关联文章图 → 才允许占位。2026-11-06 期实测 28/28 缩略图 + 5 横幅命中率 100%，总大小 ~950KB。
- 月报的资讯流缩略图同样照此办理（旧期用占位是历史状态，不要复刻）。

### 图-文相关性校验（硬要求，2026-11-06 用户反馈「要保证图像和内容有关系」后的必经工序）

「文章里有图」≠「图和这条卡片讲的是同一件事」——首图经常是表情包/影视梗/文章开头的反面铺垫图。出刊前必须逐图校验：

1. **VLM 独立描述**：全部内嵌图逐张缩到 512px 交给一个可用的视觉模型，prompt「一句话:图里具体是什么(图表/截图/产品照/示意图/表情包)?主体与可见文字?」。选择当前环境可用且获授权的视觉模型；不得在项目中保存密钥。
2. **比对换图**：描述与卡片主题不符的槽位，用「**图片在正文中的位置上下文**」选替换候选——把文章 HTML 按 `<img>` 切段，打印每张图前后各 ~60 字正文，选紧跟主题句的图（如 ApolloWorld 文中紧贴「跑一天抵三年」的虚拟路测帧，而非开头的真实路测困境图）；候选再送 VLM 复验一次。
3. **典型误配案例**（首图踩过的坑）：Orilo 卡首图是黑白猫表情包→换紧贴产品场景的产品照；主线5 首图是影视表情截图→换漏洞研究流程图；「△AI制图」「△图片由AI生成」标注的图是装饰图，不能当内容图。
4. **注意**：审计 URL 要复现当初的选择逻辑（第一张 ≥260px 的 qbitai 图），不是字面上的第一张 URL——否则会审错图。校验「无重复图」必须比对**完整 base64**，前 300 字符相同只是同 CDN 同编码参数的 JPEG 头同构，不是重复。

### 深读块 · 单事件特写（2026-11-06 新增）

用户对某张卡（通常是 Top 卡）追问「详细解释」，随后说「更新到报告里」时：把详解并入**对应卡 `.why` 之后**的 `.deep` 块——**不新增卡片，nav 覆盖数不变**。

- **结构**：`<div class="deep">` → `<span class="deeptag">深读 · 本周特写</span>` → 2–4 个 `<h5>` 小节（①②③④），小节内可含：对比 `<table>`（型号/参数对照）、参数配置 `.code` 块（Consolas 深底等宽）、`<figure class="deepfig">` 插图（640px，每小节 ≤1 张，图注带「(原文图)」）
- **CSS**：`.fcard .deep` 一族——`border-top:1px dashed` 分隔、表格 `border-collapse` + 半透明边、`.code` 深底等宽、`.deepfig img{max-width:640px}`、`figcaption` 11.5px
- **插图同样过第二关审计**：优先取「图片在正文中恰好位于该小节对应段落处」的图（如 GPT-Image-2.6 原文的「Change this to cobalt」图就在 complexity=low 工作流段落旁），再 VLM 复验
- **内容红线：事实与推断显式分开**——原文数字照抄；机理推断要么不进报告，要么显式标注「推断/官方未公开」；对用户自己管线的私人建议不进编辑内容
- **实例**：2026-11-06 期 TOP1 GPT-Image-2.6 深读（双型号表 + complexity/quality 代码块 + 3 张审计图），41 张图共 1105KB
- 插入机械：锚定该卡 `.why` 结尾文本，Python 定位 `</div>` 后插入整块；CSS 追加在 `.fcard .why b` 规则之后

## 11. 模板索取

最快路径：**复制上一期 HTML 改内容**（上期成品即权威模板——CSS、卡片骨架、JS 全在）；本文档结构与规则用于保证内容质量与一致性。
