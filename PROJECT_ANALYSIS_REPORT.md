# World Monitor 项目深度分析报告

## 项目概述

**World Monitor** 是一个实时全球情报仪表板，集成了AI驱动的新闻聚合、地缘政治监控和基础设施追踪功能。项目采用 AGPL-3.0-only 开源许可证，使用 TypeScript + Vite + deck.gl 构建，支持 Web、桌面应用（Tauri）和 PWA 多种形态。

---

## 一、技术架构分析

### 1.1 核心技术栈

| 层级 | 技术选型 | 版本/说明 |
|------|----------|-----------|
| **前端框架** | TypeScript + Vite | ES Modules, HMR 热更新 |
| **地图渲染** | deck.gl + MapLibre GL JS | WebGL 加速，3D 地球仪 |
| **后端 API** | Vercel Edge Functions | Serverless，全球 CDN 部署 |
| **协议定义** | Protocol Buffers (proto3) | 强类型 API 契约 |
| **数据缓存** | Upstash Redis | 跨用户缓存，地理索引 |
| **桌面应用** | Tauri (Rust) | 轻量级，跨平台 |
| **状态管理** | 自定义事件总线 | 无 Redux/Vuex 依赖 |

### 1.2 变体架构

项目采用**单一代码库，多形态输出**架构：

| 变体 | 环境变量 | 专注领域 | 特殊图层 |
|------|----------|----------|----------|
| **World Monitor** | `VITE_VARIANT=full` | 地缘政治、军事 | 冲突区、军事基地 |
| **Tech Monitor** | `VITE_VARIANT=tech` | 科技生态 | 科技公司 HQ、初创企业 |
| **Finance Monitor** | `VITE_VARIANT=finance` | 金融市场 | 交易所、央行、商品中心 |

---

## 二、数据源深度分析

### 2.1 军事基地数据

**数据来源**：

| 数据源 | 条目数 | 类型 | 可信度 |
|--------|--------|------|--------|
| **PizzInt** | ~79,000 | 维基百科分类数据 | 中高 |
| **OpenStreetMap** | ~53,000 | OSM 军事标记 | 中 |
| **MIRTA** | ~832 | 精选军事设施 | 高 |
| **人工整理** | ~224 | 海外基地数据 | 高 |

**数据处理流程**：

```
原始数据 → 去重（200米阈值）→ 分类（国家/类型）→ 分级（Tier 1-3）→ 最终数据集
```

**数据字段**：
- `id`: 唯一标识
- `name`: 基地名称（可能为"Unknown"）
- `lat/lon`: 坐标
- `kind`: 类型（base/airfield/naval_base等）
- `countryIso2`: 所属国家
- `type`: 分类（us-nato/china/russia/uk/france/india等）
- `tier`: 重要性级别（1=主要基地，2=一般，3=辅助设施）
- `status`: 状态（active/controversial/planned）

**存储方式**：
- **Redis GEOADD**: 地理空间索引，支持半径查询
- **Redis Hash**: 元数据存储
- **版本控制**: 原子切换，零停机更新

### 2.2 新闻数据（RSS 聚合）

**数据源规模**：
- **150+ RSS 源**（主版本）
- **80+ 源**（科技版）
- **多语言支持**: 中英两种语言的本地化源

**核心源分类**：

| 类别 | 代表源 | 语言 |
|------|--------|------|
| 地缘政治 | Reuters, BBC, Al Jazeera | 英文 |
| 防务军事 | Janes, Defense News, Breaking Defense | 英文 |
| 能源 | Energy Intelligence, OilPrice | 英文 |
| 科技 | TechCrunch, The Verge, Ars Technica | 英文 |
| 金融 | Bloomberg, FT, WSJ | 英文 |
| 中文 | 新华社、环球时报、财新 | 中文 |

**数据处理**：
- **服务端聚合**: `listFeedDigest` RPC 批量获取
- **缓存策略**: Redis 15 分钟缓存
- **去重机制**: 标题哈希去重
- **AI 分类**: 关键词 + LLM 混合分类
- **地理标注**: 从标题提取地点实体

### 2.3 实时数据流

#### 2.3.1 地震数据（USGS）

**API**: USGS GeoJSON Feed
**更新频率**: 实时
**数据字段**: 震级、深度、位置、时间
**阈值**: M4.5+ 全球地震

#### 2.3.2 军事飞行追踪（OpenSky Network）

**API**: OpenSky REST API
**数据内容**: ICAO24、呼号、位置、高度、速度
**识别逻辑**: 
- 呼号前缀匹配（如"RCH"=美国空军）
- ICAO24 代码段匹配（国家/运营商）
- 机型代码匹配

**军事运营商识别**：
```typescript
const MILITARY_CALLSIGNS = {
  'RCH': { operator: 'USAF', country: 'US' },      // Reach
  'CFC': { operator: 'RCAF', country: 'CA' },      // Canforce
  'RRR': { operator: 'RAF', country: 'UK' },       // Ascot
  'NAF': { operator: 'USN', country: 'US' },       // Navy
  // ... 更多
};
```

#### 2.3.3 船舶追踪（AIS）

**数据源**: [AISStream.io](https://aisstream.io/)
- **类型**: WebSocket 实时数据流
- **API Key**: 需要注册获取（`AISSTREAM_API_KEY`）
- **注册地址**: https://aisstream.io/authenticate

**覆盖范围**: 欧洲/大西洋较强，中东/亚洲/远洋有限（依赖地面接收站）
**数据字段**: MMSI、位置、航向、速度、船舶类型
**战略港口**: 61 个监控港口

#### 2.3.4 GPS 干扰监测

**数据源**: [GPSJam.org](https://gpsjam.org/)
- **技术**: ADS-B 应答机分析
- **网格**: H3 六边形网格
- **分类**: 无干扰/轻度/中度/重度

### 2.4 基础设施数据

#### 2.4.1 海底电缆

**数据来源**: 
- **TeleGeography** 行业数据（主要来源）
- **NGA 航行警告**: 电缆故障和维修状态
- **人工整理**: 55 条主要电缆路线

**数据文件**: `src/config/geo.ts` - `UNDERSEA_CABLES` 数组

**数据内容**: 
- 55 条主要电缆路线
- 登陆点坐标
- 容量（Tbps）、投产年份
- 所有者（Microsoft、Google、Meta 等）
- 电缆健康状态（故障、维修）
- 维修船只位置

#### 2.4.2 管道数据

**数据来源**: 
- **Global Energy Monitor** 公开数据
- **EIA** 能源信息署
- **公开地理数据**

**数据文件**: `src/config/pipelines.ts`

**覆盖**: 88 条运营中的油气管道
**分类**: 石油管道、天然气管道

#### 2.4.3 AI 数据中心

**数据来源**: [Epoch AI GPU Clusters Dataset](https://epoch.ai/data/gpu-clusters)
- **许可证**: Creative Commons Attribution
- **筛选条件**: >1,000 GPUs、Existing/Planned 状态、Confirmed/Likely 可信度

**覆盖**: 111 个主要 AI 计算集群（≥10,000 GPUs）
**字段**: 
- `name`: 数据中心名称（如 "xAI Colossus Memphis"）
- `owner`: 运营商（OpenAI、Microsoft、Meta、xAI、Amazon 等）
- `country`: 所在国家
- `lat/lon`: 坐标
- `status`: 状态（existing/planned）
- `chipType`: 芯片类型（NVIDIA GB200/H100、Amazon Trainium2 等）
- `chipCount`: GPU 数量
- `powerMW`: 功耗（兆瓦）
- `sector`: 性质（Public/Private）

### 2.5 经济数据

#### 2.5.1 市场数据

| 数据源 | 提供商 | 内容 | API Key 配置 |
|--------|--------|------|--------------|
| 股票价格 | [Finnhub](https://finnhub.io/) | 全球 92 个交易所 | `FINNHUB_API_KEY` - https://finnhub.io/register |
| 加密货币 | [CoinGecko](https://www.coingecko.com/) | BTC, ETH, SOL 等 | 免费版无需 Key |
| ETF 流向 | Yahoo Finance | 现货比特币 ETF | 无需配置 |
| 恐惧贪婪指数 | [Alternative.me](https://alternative.me/crypto/fear-and-greed-index/) | 市场情绪 | 无需配置 |

#### 2.5.2 宏观经济

| 指标 | 来源 | API Key 配置 |
|------|------|--------------|
| 美联储数据 | [FRED API](https://fred.stlouisfed.org/) | `FRED_API_KEY` - https://fred.stlouisfed.org/docs/api/api_key.html |
| 能源数据 | [EIA API](https://www.eia.gov/opendata/) | `EIA_API_KEY` - https://www.eia.gov/opendata/register.php |
| 油价 | Yahoo Finance | 无需配置 |
| 政府支出 | [USASpending.gov](https://www.usaspending.gov/) | 无需配置 |

#### 2.5.3 互联网中断监测

**数据源**: [Cloudflare Radar](https://radar.cloudflare.com/)
- **API Token**: `CLOUDFLARE_API_TOKEN` - https://dash.cloudflare.com/profile/api-tokens
- **内容**: 全球互联网中断事件

### 2.6 社会动荡与冲突数据

#### 2.6.1 ACLED (Armed Conflict Location & Event Data)

**数据源**: [ACLED](https://acleddata.com/)
- **API Key**: `ACLED_ACCESS_TOKEN` - https://developer.acleddata.com/
- **内容**: 武装冲突、抗议、暴力事件
- **覆盖**: 全球实时冲突数据

#### 2.6.2 UCDP (Uppsala Conflict Data Program)

**数据源**: [UCDP](https://ucdp.uu.se/)
- **API Key**: `UCDP_ACCESS_TOKEN` - https://ucdp.uu.se/apidocs/
- **内容**: 冲突事件、战斗相关死亡数据
- **学术机构**: 乌普萨拉大学

#### 2.6.3 GDELT

**数据源**: [GDELT Project](https://www.gdeltproject.org/)
- **内容**: 全球新闻事件数据库
- **用途**: 情报分析、正面新闻筛选
- **无需 API Key**

### 2.7 自然灾害数据

#### 2.7.1 地震数据 (USGS)

**数据源**: [USGS Earthquake API](https://earthquake.usgs.gov/fdsnws/event/1/)
- **API 地址**: `https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/4.5_day.geojson`
- **内容**: M4.5+ 全球地震
- **更新频率**: 实时
- **无需 API Key**

#### 2.7.2 GDACS (全球灾害预警系统)

**数据源**: [GDACS](https://www.gdacs.org/)
- **API 地址**: `https://www.gdacs.org/gdacsapi/api/events/geteventlist/MAP`
- **内容**: 地震、洪水、风暴、干旱、海啸
- **无需 API Key**

#### 2.7.3 NASA FIRMS (火灾监测)

**数据源**: [NASA FIRMS](https://firms.modaps.eosdis.nasa.gov/)
- **API Key**: `NASA_FIRMS_API_KEY` - https://firms.modaps.eosdis.nasa.gov/api/area/
- **内容**: 卫星热异常检测（野火）
- **卫星**: MODIS、VIIRS

### 2.8 贸易数据

**贸易航线**: 19 条全球主要航线（人工整理）
**数据文件**: `src/config/trade-routes.ts`

**分类**:
- 集装箱航线（上海-鹿特丹等）
- 能源航线（波斯湾-欧洲/亚洲）
- 散货航线（巴西-中国铁矿石）

**战略要道**: 9 个关键节点
- 马六甲海峡、霍尔木兹海峡、苏伊士运河
- 巴拿马运河、直布罗陀海峡、曼德海峡等

---

## 附录：API Key 配置汇总

### 必需配置（核心功能）

| 环境变量 | 数据源 | 注册地址 | 用途 |
|----------|--------|----------|------|
| `FINNHUB_API_KEY` | Finnhub | https://finnhub.io/register | 股票市场数据 |
| `FRED_API_KEY` | FRED | https://fred.stlouisfed.org/docs/api/api_key.html | 美联储经济指标 |
| `EIA_API_KEY` | EIA | https://www.eia.gov/opendata/register.php | 能源数据 |

### 可选配置（增强功能）

| 环境变量 | 数据源 | 注册地址 | 用途 |
|----------|--------|----------|------|
| `AISSTREAM_API_KEY` | AISStream | https://aisstream.io/authenticate | 船舶 AIS 追踪 |
| `OPENSKY_CLIENT_ID` | OpenSky | https://opensky-network.org/login?view=registration | 军事飞行追踪 |
| `OPENSKY_CLIENT_SECRET` | OpenSky | https://opensky-network.org/login?view=registration | 军事飞行追踪 |
| `WINGBITS_API_KEY` | Wingbits | https://wingbits.com/register | 飞行数据增强 |
| `NASA_FIRMS_API_KEY` | NASA FIRMS | https://firms.modaps.eosdis.nasa.gov/api/area/ | 野火监测 |
| `ACLED_ACCESS_TOKEN` | ACLED | https://developer.acleddata.com/ | 冲突事件数据 |
| `UCDP_ACCESS_TOKEN` | UCDP | https://ucdp.uu.se/apidocs/ | 冲突数据 |
| `CLOUDFLARE_API_TOKEN` | Cloudflare | https://dash.cloudflare.com/profile/api-tokens | 互联网中断监测 |

### AI 功能配置

| 环境变量 | 数据源 | 注册地址 | 用途 |
|----------|--------|----------|------|
| `GROQ_API_KEY` | Groq | https://console.groq.com/keys | AI 摘要（云端） |
| `OPENROUTER_API_KEY` | OpenRouter | https://openrouter.ai/settings/keys | AI 摘要（备选） |
| `OLLAMA_API_URL` | Ollama | https://ollama.com/download | 本地 LLM |

### 威胁情报配置

| 环境变量 | 数据源 | 注册地址 | 用途 |
|----------|--------|----------|------|
| `URLHAUS_AUTH_KEY` | URLhaus | https://auth.abuse.ch/ | 恶意 URL 情报 |
| `OTX_API_KEY` | AlienVault OTX | https://otx.alienvault.com/ | 威胁情报 |
| `ABUSEIPDB_API_KEY` | AbuseIPDB | https://www.abuseipdb.com/login | IP 威胁情报 |

---

## 三、地图渲染技术分析

### 3.1 渲染架构

**双模式渲染**：

| 模式 | 技术 | 适用场景 |
|------|------|----------|
| **桌面/高性能** | deck.gl + WebGL | 60fps，大规模数据 |
| **移动端/低性能** | D3.js + SVG | 兼容性好，低功耗 |

### 3.2 图层系统

**40+ 可切换数据图层**：

```typescript
interface MapLayers {
  hotspots: boolean;        // 情报热点
  conflicts: boolean;       // 冲突区
  bases: boolean;           // 军事基地
  nuclear: boolean;         // 核设施
  cables: boolean;          // 海底电缆
  pipelines: boolean;       // 管道
  datacenters: boolean;     // AI 数据中心
  tradeRoutes: boolean;     // 贸易航线
  ais: boolean;             // 船舶 AIS
  flights: boolean;         // 军事飞行
  protests: boolean;        // 社会动荡
  natural: boolean;         // 自然灾害
  weather: boolean;         // 天气警报
  // ... 更多
}
```

### 3.3 聚合算法（Supercluster）

**聚合逻辑**：

```typescript
// 军事基地聚合示例
private createBasesClusterLayer(): Layer[] {
  const scatterLayer = new ScatterplotLayer({
    getRadius: (d) => Math.max(8000, Math.log2(d.count) * 6000),
    getFillColor: (d) => this.getBaseColor(d.dominantType, a),
  });

  const textLayer = new TextLayer({
    getText: (d) => String(d.count),  // 显示聚合数量
  });
}
```

**聚合触发条件**：
- 缩放级别 < 5: 强制聚合
- 数据点 > 200: 自动聚合
- 单元格大小: 5° → 2° → 0.5°（随缩放递减）

### 3.4 航线渲染问题

**当前实现缺陷**：

```typescript
// 使用 ArcLayer + greatCircle
return new ArcLayer({
  greatCircle: true,  // 大圆航线
  // 问题：弧线"飘在空中"，不经过中间点
});
```

**预期 vs 实际**：
- **预期**: 上海 → 马六甲 → 曼德 → 苏伊士 → 鹿特丹
- **实际**: 弧线直接连接上海-鹿特丹，中间点仅显示为独立标记

**改进建议**: 改用 PathLayer 绘制折线，或分段绘制降低弧线高度。

---

## 四、AI 功能技术分析

### 4.1 LLM 调用链

**四级回退机制**：

```
Ollama (本地) → Groq (云端) → OpenRouter (云端) → Browser T5 (本地)
   ↓                ↓                ↓                   ↓
 5秒超时         5秒超时          5秒超时            本地执行
```

**提供商实际配置**（代码硬编码）：

| 提供商 | 实际使用模型 | API 端点 | 配置方式 |
|--------|-------------|----------|----------|
| **Ollama** | `llama3.1:8b`（可配置） | `OLLAMA_API_URL/v1/chat/completions` | 环境变量 `OLLAMA_API_URL` |
| **Groq** | `llama-3.1-8b-instant` | `https://api.groq.com/openai/v1/chat/completions` | 环境变量 `GROQ_API_KEY` |
| **OpenRouter** | `openrouter/free` | `https://openrouter.ai/api/v1/chat/completions` | 环境变量 `OPENROUTER_API_KEY` |
| **Browser T5** | `t5-small` | 浏览器本地运行 | 无需配置 |

**API Key 存储位置**：
- **服务端**: 环境变量（`process.env.GROQ_API_KEY`, `process.env.OPENROUTER_API_KEY`）
- **前端**: 不存储，通过服务端 RPC 调用
- **用户设置**: 浏览器 LocalStorage（仅本地 Ollama URL）

**注意**：
- Groq 默认使用 `llama-3.1-8b-instant`（不是 70B 版本）
- OpenRouter 默认使用 `openrouter/free` 免费模型
- Ollama 模型可通过 `OLLAMA_MODEL` 环境变量自定义

### 4.2 AI 功能模块

#### 4.2.1 世界简报（World Brief）

**功能**: LLM 综合全球头条生成摘要

**原新闻素材获取流程**:

1. **RSS 源配置** (`src/config/feeds.ts`)
   - 主版本: 150+ RSS 源
   - 科技版: 80+ RSS 源
   - 多语言: 17 种语言的本地化源

2. **源分级系统** (Source Tiers)
   | 级别 | 类型 | 代表源 |
   |------|------|--------|
   | Tier 1 | 通讯社 | Reuters, AP, AFP, Bloomberg |
   | Tier 2 | 主流媒体 | BBC, Guardian, CNN, Al Jazeera |
   | Tier 3 | 专业媒体 | Defense One, Janes, Foreign Policy |
   | Tier 4 | 聚合器 | Hacker News, Yahoo Finance |

3. **获取流程** (`src/services/rss.ts`)
   ```
   RSS 源列表 → 分批获取（每批5个）→ DOMParser 解析 → 提取标题/链接/时间
   ```
   - 每源最多取 5 条最新
   - 缓存 TTL: 30 分钟（内存）+ 持久化缓存
   - 失败冷却: 连续 2 次失败则冷却 5 分钟

4. **威胁分类**
   - 即时分类: 关键词匹配（`< 1ms`）
   - 异步分类: LLM 分类（1-5 秒，仅对关键词匹配项）
   - 分类类别: 军事冲突、网络威胁、核问题、经济制裁、自然灾害、社会动荡

**新闻筛选条件**:

1. **基础筛选**
   - 至少 2 个不同来源报道，或
   - 标记为 Alert（critical/high 威胁级别），或
   - 传播速度为 elevated/viral，或
   - 重要性评分 > 100（关键事件绕过来源要求）

2. **来源多样性限制**
   - 同一来源最多 3 条
   - 总计选择 Top 8

3. **最终选择 Top 5** 送入 LLM

**工作流程**:
1. **新闻筛选**: 从聚类事件中选择 Top 5（基于重要性评分算法）
2. **上下文构建**: 
   - 注入地理信号关联（focal points + signal convergence）
   - 注入战区态势（军事飞行数据）
3. **LLM 调用**: 通过 `generateSummary()` 调用服务端 RPC
4. **缓存**: Redis 24 小时，相同标题跨用户共享

**提示词模板**:
```
Current date: {YYYY-MM-DD}. Provide geopolitical context appropriate for the current date.

Summarize the single most important headline in 2 concise sentences MAX (under 60 words total).
Rules:
- Each numbered headline below is a SEPARATE, UNRELATED story
- Pick the ONE most significant headline and summarize ONLY that story
- NEVER combine or merge people, places, or facts from different headlines into one sentence
- Lead with WHAT happened and WHERE - be specific
- NEVER start with "Breaking news", "Good evening", "Tonight", or TV-style openings
- Start directly with the subject of the chosen headline
- If intelligence context is provided, use it only if it relates to your chosen headline
- No bullet points, no meta-commentary, no elaboration beyond the core facts

Headlines:
1. {headline_1}
2. {headline_2}
3. {headline_3}
...

Intelligence Context:
{geo_context}
```

**重要性评分算法**:
- 军事关键词: +80 基础分（war, missile, troops, airstrike...）
- 暴力/伤亡: +100 基础分（killed, dead, massacre...）
- 社会动荡: +70 基础分（protest, riot, coup...）
- 地缘热点: +60 基础分（Iran, Russia, China, Taiwan...）
- 组合加成: 暴力+热点 = 1.5x 倍率
- 商业降权: CEO/earnings/stock 关键词 = 0.3x 惩罚
- 时效性衰减: 12 小时内线性衰减（最大 0.5x）
- 传播速度加成: viral(3x) / spike(2.5x) / elevated(1.5x)

#### 4.2.2 AI 推演（Deduction）

**功能**: 地缘政治预测分析

**工作流程**:
1. **用户输入**: 查询问题（如"未来24小时中东会发生什么？"）
2. **上下文注入**: 自动追加最近 15 条新闻作为背景
3. **LLM 调用**: 调用 `IntelligenceService.deductSituation()` RPC
4. **输出**: 时间线推演 + 二阶影响分析
5. **缓存**: Redis 1 小时

**提示词模板**:
```
System Prompt:
You are a senior geopolitical intelligence analyst and forecaster.
Your task is to DEDUCT the situation in a near timeline (e.g. 24 hours to a few months) based on the user's query.
- Use any provided geographic or intelligence context.
- Be highly analytical, pragmatic, and objective.
- Identify the most likely outcomes, timelines, and second-order impacts.
- Do NOT use typical AI preambles (e.g., "Here is the deduction", "Let me see").
- Format your response in clean markdown with concise bullet points where appropriate.

User Prompt:
{user_query}

### Current Intelligence Context
{geo_context}
{recent_news}
```

**模型配置**:
- 默认模型: `llama-3.1-8b-instant`
- 温度: 0.3
- 最大 token: 1500
- 超时: 120 秒

#### 4.2.3 标题记忆（Headline Memory / RAG）

**技术栈**: 
- 嵌入模型: `all-MiniLM-L6-v2` (ONNX)
- 向量维度: 384 (float32)
- 存储: IndexedDB
- 容量: 5,000 向量（LRU 淘汰）

**工作流程**：
```
RSS 标题 → Web Worker 嵌入 → IndexedDB 存储 → 语义检索 → 相似度排序
```

### 4.3 威胁分类

**混合分类策略**：

| 阶段 | 方法 | 延迟 |
|------|------|------|
| 即时 | 关键词匹配 | < 1ms |
| 异步 | LLM 分类 | 1-5s |

**分类类别**：
- 军事冲突、网络威胁、核问题
- 经济制裁、自然灾害、社会动荡

---

## 五、后端与缓存架构

### 5.1 API 架构

**Protocol Buffers 优先**：

```protobuf
// 示例：军事服务定义
service MilitaryService {
  rpc ListMilitaryBases(ListMilitaryBasesRequest) returns (ListMilitaryBasesResponse);
  rpc ListMilitaryFlights(ListMilitaryFlightsRequest) returns (ListMilitaryFlightsResponse);
  rpc GetTheaterPosture(GetTheaterPostureRequest) returns (GetTheaterPostureResponse);
}
```

**代码生成**：
- 客户端: `src/generated/client/`
- 服务端: `src/generated/server/`
- OpenAPI: `docs/api/*.openapi.json`

### 5.2 缓存策略

**多级缓存体系**：

| 层级 | 技术 | TTL | 用途 |
|------|------|-----|------|
| **内存** | JavaScript Map | 5-15 分钟 | 组件级状态 |
| **持久化** | IndexedDB | 30 分钟 | 跨会话缓存 |
| **Redis** | Upstash | 10-60 分钟 | 跨用户缓存 |
| **CDN** | Vercel Edge | 1-24 小时 | 静态资源 |

**地理查询缓存**：
```typescript
// 网格量化缓存键
const qBbox = quantizeBbox(swLat, swLon, neLat, neLon, zoom);
const cacheKey = `military:bases:v1:${qBB}:${zoom}:${filters}:${version}`;
```

### 5.3 熔断器模式

```typescript
const breaker = createCircuitBreaker({
  name: 'Military Flight Tracking',
  maxFailures: 3,
  cooldownMs: 5 * 60 * 1000,  // 5 分钟冷却
  cacheTtlMs: 10 * 60 * 1000,  // 10 分钟缓存
});
```

---

## 六、国际化实现

### 6.1 语言支持

**2 种语言**：
- 欧洲: en
- 亚洲: zh

### 6.2 实现方式

**i18next + 懒加载**：
```typescript
// 语言包按需加载
const resources = {
  en: () => import('@/locales/en.json'),
  zh: () => import('@/locales/zh.json'),
  // ...
};
```
### 6.3 本地化 RSS 源

**自动语言匹配**：
- 浏览器语言为法语 → 自动加载 Le Monde、France24
- 浏览器语言为中文 → 加载新华社、环球时报