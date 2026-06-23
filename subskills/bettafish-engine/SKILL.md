---
name: bettafish-engine
description: |
  BettaFish 核心舆情分析引擎 — 被 bettafish-opinion-analysis MetaSKILL 作为 agent 步骤调用。
  基于 QueryAgent + MediaAgent + InsightAgent 三引擎并行架构，通过 ForumEngine 实现 Agent 间
  协作讨论，执行 3 轮反思循环优化分析结果。

  本 Skill 不直接面向用户触发，由外层 MetaSKILL 编排调用。
  输入：用户原始查询、选定的报告模板类型、提取的关键词
  输出：结构化的舆情分析发现（情感分布、平台统计、关键事件、风险、关键词等）

  **不使用任何数据库和模拟数据**，所有数据通过 WebSearch/WebFetch/Browser/Curl 实时获取。
compatibility: |
  - 需要网络搜索能力（WebSearch/WebFetch/Browser/Curl）
  - 支持主流社交媒体平台：微博、小红书、抖音、B站、知乎等
  - 无数据库依赖
  - 视频分析依赖 video-frames subskill
---

# BettaFish 核心分析引擎

BettaFish 分析引擎是一个**无数据库依赖、无模拟数据**的多智能体舆情分析引擎，采用 **QueryAgent + MediaAgent + InsightAgent 三引擎并行架构**，通过 **ForumEngine** 实现 Agent 间协作讨论，执行 **3 轮反思循环**优化分析结果。

## Agent 职责分工

| Agent | 职责 | 数据获取方式 | LLM 推荐 |
|-------|------|-------------|----------|
| **QueryAgent** | 网页搜索、新闻资讯、论坛讨论 | WebSearch + WebFetch + Browser | DeepSeek |
| **MediaAgent** | 短视频/图文内容分析 | 下载视频 → video-frames 提取帧 → 分析 | Gemini-2.5-pro |
| **InsightAgent** | 情感分析、关键词提取、聚类分析 | WebSearch + Python 脚本分析 | Kimi-K2 |
| **ForumHost** | 主持 Agent 讨论、引导研究方向 | LLM 生成 | Qwen-Plus |

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

### Phase 1: 并行启动三 Agent（Round 1 — 初步搜索）

**所有 Agent 同时启动，独立执行：**

#### QueryAgent 执行流程：
1. 使用 WebSearch 搜索新闻/论坛/博客（使用 `keywords.query_agent`）
2. 使用 Browser 访问关键页面获取详细内容
3. 使用 WebFetch 获取页面结构化数据
4. 提取关键信息，生成初步摘要
5. **发布发现到 ForumEngine**（按 ForumEngine 协作协议格式）

#### MediaAgent 执行流程：
1. 使用 WebSearch 搜索短视频/图文内容链接（使用 `keywords.media_agent`）
2. 下载相关视频文件到本地临时目录
3. 使用 **video-frames subskill** 提取关键帧：
   ```bash
   {baseDir}/subskills/video-frames/scripts/frame.sh /tmp/video.mp4 --time 00:00:05 --out /tmp/frame_5s.jpg
   ```
4. 分析视频帧内容（画面元素、文字、场景）
5. 结合视频描述文字进行综合分析
6. **发布发现到 ForumEngine**

#### InsightAgent 执行流程：
1. 使用 WebSearch 获取补充数据（使用 `keywords.insight_agent`）
2. 执行情感分析（基于规则引擎）：
   ```python
   from scripts.sentiment_analyzer import SentimentAnalyzer
   analyzer = SentimentAnalyzer()
   results = analyzer.analyze_batch(texts)
   ```
3. 提取关键词和热点话题
4. 使用 **search_clustering.py** 对搜索结果聚类采样：
   ```python
   from scripts.search_clustering import cluster_search_results, sample_from_clusters
   clusters = cluster_search_results(results=search_results, n_clusters=5, samples_per_cluster=3)
   ```
5. **发布发现到 ForumEngine**

### Phase 2: ForumHost 第 1 轮引导

ForumHost 收集到 3 条 Agent 发言后：
1. 总结本轮发现要点
2. 识别信息缺口和矛盾点
3. 提出第 2 轮研究方向（`directions`）
4. 抛出需要回答的关键问题（`questions`）

### Phase 3: 三 Agent 深度搜索（Round 2）

三个 Agent 根据 ForumHost 第 1 轮引导进行针对性深度搜索，再次发布发现到 ForumEngine。

### Phase 4: ForumHost 第 2 轮引导

ForumHost 总结前两轮发现，聚焦核心矛盾，提出第 3 轮验证方向。

### Phase 5: 三 Agent 验证搜索（Round 3）

三个 Agent 交叉验证前两轮发现，确认关键结论，消除信息矛盾。

### Phase 6: ForumHost 最终总结

ForumHost 综合三轮讨论结果，生成结构化分析输出。

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

## ForumEngine 协作协议

### Agent 发言格式

每个 Agent 完成一轮搜索后，按以下格式发布发现：

```json
{
  "agent": "QueryAgent",
  "round": 1,
  "findings": {
    "key_points": ["发现1", "发现2"],
    "sources": [{"title": "...", "url": "..."}],
    "confidence": 0.85
  },
  "questions": ["需要进一步验证的问题"]
}
```

### ForumHost 引导格式

ForumHost 每收集到 3 条 Agent 发言后生成引导：

```json
{
  "host": "ForumHost",
  "round": 1,
  "summary": "本轮讨论总结",
  "directions": ["下一步研究方向1", "方向2"],
  "questions": ["需要回答的关键问题"]
}
```

## 数据采集规范

### QueryAgent 数据获取

```
1. WebSearch("关键词 + 口碑评价")
2. WebSearch("site:weibo.com 关键词")
3. Browser 访问关键页面获取详细内容
4. WebFetch 获取页面结构化数据
5. Curl 命令行获取 API 数据或特殊页面
```

Curl 使用示例：
```bash
curl -s -H "User-Agent: Mozilla/5.0" -H "Accept: application/json" "https://api.example.com/data"
```

### MediaAgent 视频分析

```
1. WebSearch 搜索视频链接（抖音、B站、小红书）
2. 下载视频到本地临时目录
3. 使用 video-frames subskill 提取关键帧
4. 分析视频帧内容（画面元素、文字、场景）
5. 结合视频描述文字进行综合分析
```

### InsightAgent 数据分析

使用项目内置 Python 脚本进行分析：

```python
# 情感分析
from scripts.sentiment_analyzer import SentimentAnalyzer
analyzer = SentimentAnalyzer()
results = analyzer.analyze_batch(texts)

# 搜索结果聚类采样
from scripts.search_clustering import cluster_search_results
clusters = cluster_search_results(search_results, n_clusters=5)

# 知识图谱构建
from scripts.graph_generator import KnowledgeGraphBuilder
builder = KnowledgeGraphBuilder(topic=topic)
builder.build_from_analysis_result(query_results, media_results, insight_results)
```

## 质量要求

- [ ] 三 Agent 并行执行完成（每轮）
- [ ] ForumEngine 完成 3 轮讨论
- [ ] 所有数据来自真实 WebSearch，无模拟数据
- [ ] 视频分析使用 video-frames 提取真实帧
- [ ] 搜索结果经过聚类采样处理
- [ ] ForumHost 每轮都输出了有效的引导方向
