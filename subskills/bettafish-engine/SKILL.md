---
name: bettafish-engine
description: |
  BettaFish 核心舆情分析引擎 — 被 bettafish-opinion-analysis MetaSKILL 作为 agent 步骤调用。
  基于 QueryAgent + MediaAgent + InsightAgent 三引擎角色扮演架构，执行 3 轮反思循环优化分析结果。

  本 Skill 不直接面向用户触发，由外层 MetaSKILL 编排调用。
  输入：用户原始查询、选定的报告模板类型、提取的关键词
  输出：结构化的舆情分析发现（情感分布、平台统计、关键事件、风险、关键词等）

  **不使用任何数据库和模拟数据**，所有数据通过 web_search / web_fetch / browser / shell(curl) 实时获取。
compatibility: |
  运行环境: OpenClaw.net AgentRuntime
  必需工具:
    - web_search (Tavily/Brave/SearXNG 后端)
    - web_fetch (HTTP 页面抓取)
    - browser (Playwright 无头浏览器)
    - shell (curl + Python 脚本执行)
    - file_read / file_write (脚本输出读写)
  可选工具:
    - video-frames skill (视频关键帧提取，同级 subskill)
  支持平台: 微博、小红书、抖音、B站、知乎等主流社交媒体
  无数据库依赖
---

# BettaFish 核心分析引擎

BettaFish 分析引擎是一个**无数据库依赖、无模拟数据**的舆情分析引擎，在 OpenClaw AgentRuntime 中以**单 Agent 角色扮演**方式模拟 QueryAgent + MediaAgent + InsightAgent 三引擎协作，执行 **3 轮反思循环**优化分析结果。

> **OpenClaw 运行时说明**：本 Skill 运行在 OpenClaw.net 的 `AgentRuntime` 中。所有工具调用（`web_search`、`web_fetch`、`browser`、`shell`）均为 OpenClaw 原生工具。三引擎并非独立 Agent 进程，而是由当前 Agent **依次扮演三个角色**，ForumHost 角色由 Agent 自身在每轮结束时总结担任。

## Agent 角色职责（角色扮演）

| 角色 | 职责 | OpenClaw 工具 | 搜索侧重 |
|------|------|--------------|---------|
| **QueryAgent** | 网页搜索、新闻资讯、论坛讨论 | `web_search` + `web_fetch` + `browser` | 新闻/博客/论坛/官方声明 |
| **MediaAgent** | 短视频/图文内容分析 | `web_search` + `shell`(curl下载) + video-frames skill | 抖音/B站/小红书/微博视频 |
| **InsightAgent** | 情感分析、关键词提取、聚类 | `web_search` + `shell`(python脚本) | 补充数据+数据挖掘 |
| **ForumHost** | 主持讨论、引导研究方向 | 角色内总结（不调用工具） | 综合分析缺口 |

## 输入规范

本 Skill 接收外层 MetaSKILL 传递的以下输入：

```yaml
input:
  user_query: "用户的原始分析请求"
  template: "brand_reputation | crisis | hotspot | competition | policy | monitoring"
  keywords:
    query_agent: ["关键词1", "关键词2"]
    media_agent: ["关键词1", "关键词2"]
    insight_agent: ["关键词1", "关键词2"]
  time_range: "近7天 | 近30天 | 自定义"
```

## 执行流程

> 每轮中，Agent 依次扮演 QueryAgent → MediaAgent → InsightAgent → ForumHost 四个角色。
> 将每轮的结果记录在内存中，供下一轮 ForumHost 分析差距。

### Phase 1: Round 1 — 初步搜索

**依次扮演三个角色，每角色独立执行搜索后记录发现：**

#### 角色一：QueryAgent
1. 调用 `web_search` 搜索新闻/论坛/博客（使用 `keywords.query_agent`）：
   ```
   web_search(query="关键词 + 口碑评价", max_results=10)
   web_search(query="site:weibo.com 关键词", max_results=10)
   ```
2. 对高价值结果调用 `web_fetch` 获取全文：
   ```
   web_fetch(url="https://...", max_length=8000)
   ```
3. 对需要JS渲染的页面调用 `browser` 访问：
   ```
   browser(action="navigate", url="https://...")
   browser(action="snapshot")  -- 获取页面可访问性树
   ```
4. 提取关键信息（事件时间线、主要观点、关键人物/机构）
5. 以结构化格式记录 QueryAgent 发现

#### 角色二：MediaAgent
1. 调用 `web_search` 搜索短视频/图文内容链接（使用 `keywords.media_agent`）：
   ```
   web_search(query="关键词 抖音 视频", max_results=10)
   web_search(query="关键词 小红书 图文", max_results=10)
   ```
2. 对可直接访问的图文页面，调用 `browser` 截图：
   ```
   browser(action="navigate", url="https://...")
   browser(action="screenshot")
   ```
3. 如有视频链接，使用 `shell` 下载后调用 **video-frames skill**（通过 `load_skill` 加载）提取帧：
   ```bash
   shell(command="curl -L -o /tmp/analysis_video.mp4 '视频URL'", timeout_seconds=60)
   ```
   然后按 video-frames skill 指令提取关键帧并分析
4. 分析视觉内容（画面元素、文字叠加、场景情绪）
5. 以结构化格式记录 MediaAgent 发现

#### 角色三：InsightAgent
1. 调用 `web_search` 获取补充数据（使用 `keywords.insight_agent`）
2. 使用 `shell` 运行情感分析脚本（先通过 `read_skill_resource` 获取脚本内容）：
   ```bash
   shell(command="python {skill_dir}/scripts/sentiment_analyzer.py --input /tmp/insight_texts.json --output /tmp/sentiment_results.json", timeout_seconds=120)
   ```
3. 提取关键词和热点话题
4. 对搜索结果聚类采样：
   ```bash
   shell(command="python {skill_dir}/scripts/data_collector.py --keywords '关键词1,关键词2' --output /tmp/collected_data.json", timeout_seconds=120)
   ```
5. 以结构化格式记录 InsightAgent 发现

#### 角色四：ForumHost（第 1 轮引导）
回顾三个角色的发现，输出：
1. 本轮发现要点总结
2. 信息缺口和矛盾点
3. 第 2 轮研究方向（`directions`）
4. 需要回答的关键问题（`questions`）

### Phase 2: Round 2 — 深度搜索

依次扮演三个角色，根据 ForumHost 第 1 轮引导进行针对性深度搜索，记录发现。

### Phase 3: ForumHost 第 2 轮引导

总结前两轮发现，聚焦核心矛盾，提出第 3 轮验证方向。

### Phase 4: Round 3 — 验证搜索

依次扮演三个角色，交叉验证前两轮发现，确认关键结论，消除信息矛盾。

### Phase 5: ForumHost 最终总结

综合三轮讨论结果，生成结构化分析输出（见下方输出规范）。

## 输出规范

分析完成后，输出以下结构化数据：

```yaml
output:
  sentiment:
    positive: 0.45        # 正面比例
    neutral: 0.30         # 中性比例
    negative: 0.25        # 负面比例
    distribution: [...]   # 情感分布详情

  platform_stats:
    weibo: {volume: 1234, sentiment: {...}}
    xiaohongshu: {volume: 567, sentiment: {...}}
    douyin: {volume: 890, sentiment: {...}}
    bilibili: {volume: 345, sentiment: {...}}
    zhihu: {volume: 234, sentiment: {...}}

  keywords:
    - {word: "关键词1", frequency: 0.85, sentiment: "positive"}
    - {word: "关键词2", frequency: 0.72, sentiment: "negative"}

  key_events:
    - title: "关键事件1"
      time: "2026-06-20"
      summary: "事件概述"
      sources: [{title: "...", url: "..."}]
      sentiment_impact: "negative"

  risks:
    - level: "high | medium | low"
      category: "产品质量 | 服务体验 | 品牌形象 | 竞争威胁"
      description: "风险描述"
      evidence: ["证据1", "证据2"]

  forum_discussion:
    - round: 1
      agent_findings: [...]
      host_guidance: {...}
    - round: 2
      agent_findings: [...]
      host_guidance: {...}
    - round: 3
      agent_findings: [...]
      host_summary: {...}

  sources:
    - {title: "...", url: "...", type: "news|social|video|forum"}
```

## ForumHost 协作协议（角色内模拟）

### Agent 角色发言格式

每个角色完成一轮搜索后，按以下 JSON 格式输出发现：

```json
{
  "agent": "QueryAgent | MediaAgent | InsightAgent",
  "round": 1,
  "findings": {
    "key_points": ["发现1", "发现2", "发现3"],
    "sources": [{"title": "来源标题", "url": "https://..."}],
    "data_quality": "high | medium | low",
    "confidence": 0.85
  },
  "questions": ["需要进一步验证的问题1", "问题2"],
  "tool_calls_summary": "本轮使用的工具及调用次数简述"
}
```

### ForumHost 引导输出格式

ForumHost 在每轮结束时（收集到三个角色发现后）生成：

```json
{
  "host": "ForumHost",
  "round": 1,
  "summary": "本轮讨论总结（3-5句话）",
  "gaps_identified": ["信息缺口1", "信息缺口2"],
  "contradictions": ["发现的矛盾点"],
  "directions": ["第N+1轮研究方向1", "方向2"],
  "questions": ["需要回答的关键问题1", "问题2"]
}
```

## 数据采集规范

### QueryAgent 数据获取（OpenClaw 工具）

```
1. web_search("关键词 + 口碑评价")                    → Tavily/Brave/SearXNG 搜索
2. web_search("site:weibo.com 关键词")                → 定向平台搜索
3. web_fetch("https://...")                           → 获取全文文本
4. browser → navigate("https://...") → snapshot()     → JS渲染页面内容
5. shell("curl -s -H 'User-Agent: Mozilla/5.0' 'https://api.example.com/data'")  → API数据
```

### MediaAgent 视频分析

```
1. web_search 搜索视频链接（抖音、B站、小红书）
2. shell(curl) 下载视频到 /tmp/ 目录
3. load_skill("video-frames") 获取视频帧提取指令
4. shell 执行 frame.sh 提取关键帧
5. 分析视频帧内容 + 描述文字
```

> **video-frames skill 依赖**：本 skill 依赖同级的 `video-frames` subskill。使用前通过 `load_skill("video-frames")` 加载其指令，然后按其说明调用 `{skill_dir}/../video-frames/scripts/frame.sh`。

### InsightAgent 数据分析

使用 `shell` 工具运行 Python 脚本（脚本位于本 skill 的 `scripts/` 目录，可通过 `read_skill_resource` 获取内容）：

```bash
# 情感分析（基于规则引擎 + 词典）
shell(command="python {skill_dir}/scripts/sentiment_analyzer.py --input /tmp/insight_texts.json --output /tmp/sentiment.json", timeout_seconds=120)

# 搜索结果聚类采样
shell(command="python {skill_dir}/scripts/data_collector.py --mode cluster --input /tmp/search_results.json --clusters 5 --samples 3", timeout_seconds=120)

# 知识图谱数据生成（供外层 build_graph 步骤使用）
shell(command="python {skill_dir}/scripts/graph_generator.py --topic '主题' --input /tmp/analysis_data.json --output /tmp/graph_data.json", timeout_seconds=120)
```

> **Python 环境要求**：脚本需要 Python 3.10+ 及 jieba、snownlp 依赖。首次使用前确保环境已配置：
> ```bash
> shell(command="pip install jieba snownlp pyyaml", timeout_seconds=60)
> ```

## 质量要求

- [ ] 三个角色（QueryAgent/MediaAgent/InsightAgent）每轮均执行完成
- [ ] ForumHost 完成 3 轮引导总结
- [ ] 所有数据来自 `web_search` / `web_fetch` / `browser` / `shell(curl)` 真实请求，无模拟数据
- [ ] 视频分析使用 video-frames skill 提取真实帧
- [ ] 搜索结果经过聚类采样处理
- [ ] ForumHost 每轮都输出了有效的引导方向
- [ ] 最终输出包含完整的情感分布、平台统计、关键事件、风险和来源清单
- [ ] 所有 `web_search` 调用结果标注来源 URL 和获取时间

## OpenClaw 集成说明

### 工具映射速查

| 本文档术语 | OpenClaw 工具名 | 后端 |
|-----------|---------------|------|
| WebSearch | `web_search` | Tavily / Brave / SearXNG |
| WebFetch | `web_fetch` | HttpClient (HTML→text) |
| Browser | `browser` | Playwright (Chromium) |
| Curl | `shell` | cmd.exe / /bin/sh |
| Python脚本 | `shell` | Python subprocess |

### Skill 资源加载

本 Skill 的 `scripts/` 目录中的 Python 脚本通过 OpenClaw 的渐进式披露机制加载：
- 先用 `read_skill_resource("bettafish-engine", "scripts/sentiment_analyzer.py")` 获取脚本内容
- 再用 `file_write` 写入临时目录，或直接 `shell` 调用 skill 目录下的脚本

### 与父 MetaSKILL 的关系

本 Skill 由 `bettafish-opinion-analysis` MetaSKILL 通过 `meta_invoke` → agent step 调用。
MetaSKILL 传入 `template` + `keywords` + `user_query`，本 Skill 返回结构化 `output` YAML。
