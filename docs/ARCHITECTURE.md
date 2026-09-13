# NextTHUxk 代码结构与代码逻辑

> 本文档基于 v2.1.1 源码（2026-09）逐文件阅读整理，描述扩展的文件结构、模块职责、
> 核心数据流与关键机制。变量/字段级别的逐一说明见 [VARIABLES.md](./VARIABLES.md)。

---

## 目录

1. [项目概览](#1-项目概览)
2. [文件结构与加载顺序](#2-文件结构与加载顺序)
3. [总体架构模式](#3-总体架构模式)
4. [启动流程（launch）](#4-启动流程launch)
5. [数据获取层](#5-数据获取层)
6. [阶段判定与两种数据模型](#6-阶段判定与两种数据模型)
7. [中签概率计算](#7-中签概率计算)
8. [课程身份与匹配仲裁](#8-课程身份与匹配仲裁)
9. [课表解析与冲突检测](#9-课表解析与冲突检测)
10. [渲染层](#10-渲染层)
11. [搜索管线（随时查询模式）](#11-搜索管线随时查询模式)
12. [暂存 / 草稿 / 提交流程](#12-暂存--草稿--提交流程)
13. [选课 / 退选 / 志愿调整 API](#13-选课--退选--志愿调整-api)
14. [AI 层](#14-ai-层)
15. [THU选课社区评价层](#15-thu选课社区评价层)
16. [更新检查](#16-更新检查)
17. [容错设计汇总](#17-容错设计汇总)

---

## 1. 项目概览

**NextTHUxk** 是一个清华本科生选课增强浏览器扩展（Manifest V3，Chrome / Edge / Firefox 通用，
Firefox 有签名 xpi）。它在原选课页面（zhjwxk / zhjw / webvpn 三个域名）上叠加一个
全屏工作台，提供：

- 课程搜索（服务端随时查询，支持北大 PK/GPK、北外 BW 外校课）
- 中签概率计算（志愿级联模型）与课余量/排队模型（按选课阶段自动切换）
- 动态时间轴课表预览 + 区间重叠制冲突检测
- 暂存课表 / 多套草稿 / 差量提交（一键退选+选入）
- 培养方案覆盖检测、THU选课社区评价（thubook.help）、AI 搜索与排课
- GitHub Releases 自动更新检查

**技术栈**：纯原生 JavaScript，无构建工具、无框架、无 npm 依赖。全部代码通过
`manifest.json` 的 `content_scripts` 按序注入页面，运行在 content script 环境
（与页面共享 DOM，但 JS 隔离）。

**代码规模**（约 6,700 行 JS）：

| 文件 | 行数 | 职责 |
|---|---|---|
| `src/render.js` | 1717 | 所有渲染函数 + 筛选/分页/服务端搜索调度 |
| `src/data.js` | 1898 | 数据抓取与解析（目录/志愿/课余量/选退课 API/服务端搜索） |
| `src/state.js` | 886 | 课表解析、冲突检测、暂存/草稿管理、选课状态 |
| `content.js` | 722 | 入口：HTML 模板 + Shadow DOM + 事件绑定 + 启动流程 |
| `src/reviews.js` | 455 | THU选课社区评价层（匹配/弹窗/联想词） |
| `src/config.js` | 349 | 命名空间、常量、工具、存储、网络层、分页抓取器 |
| `src/ai.js` | 273 | AI 课程搜索 + 智能排课 |
| `src/probability.js` | 218 | 中签概率计算、志愿格式化 |
| `src/update.js` | 106 | 版本更新检查 |
| `src/gbk.js` | 36 | GBK 查询参数编码表 |
| `popup.js` | 30 | 扩展弹出页逻辑 |

---

## 2. 文件结构与加载顺序

```
NextTHUxk/
├── manifest.json          # MV3 配置：权限、host、content_scripts 注入声明
├── content.js             # 入口：HTML 模板 + Shadow DOM + 事件绑定 + launch 启动流程
├── content.css            # 全部样式（液态玻璃 UI，通过 web_accessible_resources 加载）
├── popup.html / popup.js  # 浏览器工具栏弹出页（状态检测 + 远程启动工作台）
├── icons/                 # 扩展图标 16/32/48/128
└── src/
    ├── config.js          # NX 命名空间、常量、NX.state、存储、网络、pagedFetch
    ├── gbk.js             # GBK 百分号编码（OneTHU gbk-table.ts 逐字节移植）
    ├── data.js            # 教务 HTML/接口解析 + 全部数据获取函数
    ├── probability.js     # 概率级联计算
    ├── reviews.js         # 社区评价（IIFE 封装，三级匹配器）
    ├── state.js           # 课表/冲突/暂存/草稿/选课状态
    ├── render.js          # 渲染 + 筛选 + 搜索调度
    ├── ai.js              # AI 搜索 + 排课
    └── update.js          # 更新检查
```

**加载顺序**（`manifest.json` `content_scripts.js` 数组，前 9 项按序执行，`run_at: document_idle`）：

```
config.js → gbk.js → data.js → probability.js → reviews.js
→ state.js → render.js → ai.js → update.js → content.js
```

依赖关系：

- `config.js` **最先**加载：创建 `var NX = window.NX = {}`，后续每个文件开头
  `var NX = NX || {}` 挂载自己的方法（同一 content script 上下文共享全局）。
- `content.js` **最后**加载：解构引用各模块函数、注入 UI、绑定事件，是唯一
  的"主动执行者"；其余模块只定义 `NX.*` 函数，不主动执行（`reviews.js` 例外，
  IIFE 内仅初始化状态不发起请求）。
- 权限：`activeTab` / `storage` / `unlimitedStorage`；host 覆盖 GitHub API、
  webvpn/zhjw/zhjwxk、thubook.help。

---

## 3. 总体架构模式

### 3.1 NX 全局命名空间单例

所有模块向同一个 `NX` 对象挂载函数与常量（如 `NX.fetchPage`、`NX.state`、`NX.ZY_LIMITS`）。
没有 class、没有模块系统、没有 import——这是刻意保持的零依赖风格。模块间调用
一律通过 `NX.*`，配合函数内解构（`const { state, fetchPage } = NX;`）获得局部引用。

### 3.2 Shadow DOM 隔离

`content.js:178-187`：创建 `#nextthuxk-host`（`all:initial; z-index:2147483647`）并
`attachShadow({ mode: 'open' })`，全部 UI（启动按钮 + 全屏工作台 + 各弹窗）都渲染在
Shadow Root 内，样式完全隔离，不受教务页面 CSS 影响。

- `state.host` / `state.shadow` 保存宿主与根节点；
- `state.$ = id => shadow.getElementById(id)` 是全项目的元素查询入口；
- `content.css` 通过 `fetch(browser.runtime.getURL('content.css'))` 读入后以
  `<style>` 注入 Shadow Root（这也是它声明在 `web_accessible_resources` 的原因）。

### 3.3 双浏览器兼容

`config.js:7`：`NX.browser = typeof browser !== 'undefined' ? browser : chrome`。
所有 storage / tabs / runtime 调用统一走 `NX.browser`。`NX.store.set` 同时兼容
Firefox 的 Promise API 与 Chrome MV3 的 callback 形式，写失败 reject + 打日志
（不再静默，v1.3.15 修大缓存写回静默失败）。

### 3.4 入口守卫

`content.js:8-9`：

```js
if (window.parent !== window) return;                    // 不在 iframe 重复注入
if (!/zhjwxk|zhjw\.cic|webvpn/.test(location.hostname)) return;
```

### 3.5 WebVPN 适配

`content.js:28-31`：WebVPN 页面 URL 形如 `/http|https/<32+位hex>/<原路径>`，
`state.BASE` 必须保留编码站点前缀，否则所有请求打到 webvpn 根目录 404。
`NX.ensureSiteIdentity`（content.js:193）用 AES-128-CBC（key=iv=`wrdvpnisthebest!`）
解密路径中的主机段，精确判定 zhjwxk / zhjwj；解密失败退回路径嗅探（`xkBks.`）。

---

## 4. 启动流程（launch）

`content.js:240` `NX.launch` 是点击启动按钮 / popup 发消息后的总入口。
`state.launching` 并发锁防双击双抓。

```
launch()
├── toggle(true)                          # 显示工作台，隐藏启动按钮
├── ensureSiteIdentity()                  # WebVPN 站点识别（#21）
├── 解析 SEM（URL p_xnxq → storage → prompt 手输）并写回 storage
├── 解析 GRADE（storage → prompt 手输，仅影响 AI 推荐）
├── 加载 manualEvents（自定义占用）
├── 读 staticData 缓存（ver 不符则清空 + 重置年级）
├── Promise.all 并行五路：               # content.js:293
│   ├── fetchSelectedCourses()            # 已选课（失败 → 一级课表兜底）
│   ├── fetchCandidateCourses()           # 候补队列（dlSearch → kbSearch 兜底）
│   ├── fetchTrainingPlan()               # 培养方案（有缓存则跳过）
│   ├── fetchLevelTable()                 # 一级课表（课程类型权威源）
│   └── fetchCategoryAttrs()              # 预选分类页属性（必修/限选 tab）
├── state.levelMap = {...levelMap, ...catAttrs}   # 分类页属性优先
├── backfillCandidateMeta(candCourses)    # 候补课逐门补学分/容量
├── 核心池构建：allCourses = 已选 + 候补  # 不含全量目录（2.x 随时查询）
├── knoteLoad()                           # 课表时间持久缓存先于首渲
├── rebuildCourseMap() + applyLevelMap(pool)
├── （非阻塞后台任务, content.js:333）:
│   ├── fetchQueueData(pool)              # 课余量/排队 → isQueuePhase 判定
│   ├── isQueuePhase ? 池行余量合并
│   │                 : fetchVolunteer(pool, {onData})  # 志愿统计增量回调
│   │                   └ 已选课缺行 → fetchVolCourse 定向补拉
│   ├── 阶段标签（"课余量模式"chip）
│   └── filterCourses / renderPreviewTT / renderQueueSection / renderStageCart
├── 学期竞态守卫后写 staticData（仅 plan）
├── renderPlan / renderPreviewTT / renderQueueSection
├── renderStageAndDrafts()                # 暂存 + 草稿载入 + baseFlag 迁移
├── backfillStageRows()                   # 暂存/草稿课不在池 → 按课号补拉（落池点亮暂存余量徽章/草稿预览合成）
├── finishLaunch(...)                     # 缓存信息、志愿定时同步、AI 配置回填、
│                                         # backfillSelTimes、社区评价索引
└── filterCourses()                       # 初始落点：浏览模式第 1 页
```

要点：

- **绝不整库预爬**（2.x 核心变化）：启动只拉"已选/候补/方案/类型"个位数到几十个请求；
  课程列表一律搜索时服务端随时查（详见 §11）。
- **非阻塞**：课余量/志愿同步是后台任务，UI 先上屏、数据后到回填。
- **SEM0 竞态基准**：学期在加载中途被切换时弃写缓存（v1.4.4）。
- 定时同步：`startVolAutoSync`（content.js:547）按教务检查点
  8/12/16/20 点调度 `syncQueueAndVol`。

---

## 5. 数据获取层

### 5.1 网络基础（config.js）

| 函数 | 作用 |
|---|---|
| `NX._fetchRaw(url, opts)` | 15s 超时（AbortController）+ 按响应头/内容判定 GBK/UTF-8 解码 |
| `NX.decodeBest(buf, url)` | 编码探测：①content-type 声明 ②替换符 U+FFFD 单侧判定 ③标签数多者胜 ④默认 GBK（`_GBK_URL_RE` 匹配的教务 URL） |
| `NX.fetchPage(url, opts)` | 在 `_fetchRaw` 上加 **WebVPN 票据自愈**：响应是壳页（`__vpn_hostname_data`）→ `reenterZhjwxk()` 重进教务入口换票（60s 冷却）→ 重试一次；SSO 登录页直接透传上层诊断 |
| `NX.fetchPageDual(url)` | 双解码抓取：同时返回 `{gbk, utf8}` 两种解码文本 |
| `NX.pickDecoded(parse, dual)` | 双解码选优：`parse(html).length ×2 + 无替换符+1`，平分回退 GBK |

编码问题是本项目的持久战场：教务老页面是 GBK，部分新接口是 UTF-8，WebVPN 代理
还会改写响应。策略是"谁解析出的行多谁赢"（`kbSearch` 的「候选：」中文标记只在
正确解码下被正则命中，天然判别器）。

`src/gbk.js` 提供查询参数方向的补码：中文筛选参数（课名/教师）必须 GBK 百分号
编码，否则教务 LIKE 匹配不到返回 0 行。实现是查表法：`GBK_CHARS`（字符表）+
`GBK_BYTES_B64`（字节表 base64），惰性建 `Map` 索引。

### 5.2 通用分页抓取器 `pagedFetch`（config.js:169）

教务几乎所有列表都是分页 HTML。`pagedFetch` 统一并发分页抓取：

- **参数**：`fetchFirst`/`firstHtml`、`fetchPage(p)`、`parse(html)→{items,hasData}`、
  `maxPages`、`concurrency`、`dedupe(item)→key`、`retry`、`expectPages`、`throttle`、`cooldown`。
- **节流闸**（`gate`）：预约式占用槽位，保证任意两次实际请求间隔 ≥ throttle。
- **重试语义**：单页失败/空页重试 `retry` 次（退避 `retryDelay×(attempt+1)`）；
  重试期间 `pause = true` 暂停铺新页（防越界请求）；仍空 → `'EMPTY'`（真末页）。
- **合并语义**（`absorb`）：`p=0` 与首页可能都有数据（0 基分页），合并不覆盖
  （v1.3.6 修复首页数据丢失）。
- **总数校验补抓**：`expectPages` 已知时，扫完后对缺失页两轮补抓（并发 2 → 冷却
  8s → 并发 1）；连续 5 页失败熔断（session 失效保护）；纯 EMPTY 免冷却
  （服务器稳定返回空页时等待无意义，v1.3.8）。
- **空页诊断**（`diagEmpty`）：记录首个空页特征（accessDenied/登录失效/gridData），
  只记一例避免刷屏。

`NX.runPool(items, concurrency, fn)` 是无终止语义的固定并发池，供逐课号查询用。
`NX.debounce` / `NX.lc`（小写化缓存 Map）为通用工具。

### 5.3 教务端点清单（data.js）

| 端点 | 用途 | 关键函数 |
|---|---|---|
| `xkBks.vxkBksJxjhBs.do?m=kkxxSearch` | 开课信息检索（服务端搜索主通道） | `serverSearch` |
| `xkBks.vxkBksXkbBs.do?m=bxSearch/xxSearch/rxSearch/tySearch` | 预选分类页签（必修/限选/任选/体育） | `fetchCategoryAttrs` / `tabSearchByKch` |
| `xkBks.vxkBksXkbBs.do?m=yxSearchTab` | 已选课列表 | `fetchSelectedCourses` |
| `xkBks.vxkBksXkbBs.do?m=dlSearch` / `dlSearchTab` | 候补队列 | `fetchCandidateCourses` / `dropCourse` |
| `xkBks.vxkBksXkbBs.do?m=kbSearch` | 一级课表（候选兜底/时间正源） | `parseTimetableCandidates` |
| `syxk.vsyxkKcapb.do?m=ztkbSearch` | 整体课表（已选/候补时间·教师正源，#46） | `fetchWholeTimetable` / `parseWholeTimetable` |
| `xkBks.vxkBksXkbBs.do?m=xkqkSearch` | 课余量首页（阶段探针） | `fetchQueueData` |
| `xkBks.vxkBksJxjhBs.do` POST `m=kylSearch` | 按课号精确课余量 | `fetchQueueData` 内 `kylPost` |
| `xkBks.vxkBksXkbBs.do?m=selectBksDlCount` | 排队人数批量（批 100） | `fetchQueueData` |
| `xkBks.xkBksZytjb.do?m=tbzySearchBR` | 志愿统计（按院系） | `fetchVolunteer` |
| `xkBks.xkBksZytjb.do?m=tbzySearchTy` | 体育志愿统计（无院系轴，全量 ≤20 页） | `fetchVolunteer` |
| `xkBks.vjhBksPyfakcbBs.do` | 培养方案 | `fetchTrainingPlan` |
| `xkBks.vxkBksXkbBs.do`（POST） | 选课/退课/志愿调整（token 表单） | `fetchFormSubmit` |
| `js.vjsKcbBs.do?m=showToXs` | 课程简介 | `fetchCourseDetail` |
| `xkBks.vxkBksXkbBs.do?pathContent=一级课表` | 课程类型权威源 | `fetchLevelTable` |

### 5.4 解析器（data.js 前半）

- `parsePlan` / `parseFullProgram`：培养方案两种页面（zhjwxk 简表 / zhjw 全方案表）。
- `parseCatalog`：kkxxSearch 结果表 → 课程对象（20 字段，含外校课号放行规则
  `/^[A-Za-z0-9]+$/ && /\d/`、行内属性扫描、说明列 `note` 即外校时间载体）。
  **学分规则**：本校课（课号纯数字）学分 = 课号最后一位（`NX.lastDigitCredits`，
  外校课返回 null 走学分列原解析）——parseCatalog / parsePlan / parseFullProgram /
  fetchSelectedCourses / parseTabGrid 及各兜底行（一级课表重建 / 候补 dlSearch /
  kbSearch）共 8 处解析源头统一套用；暂存/草稿持久化快照在 `renderStageAndDrafts`
  内由 `NX.migrateStageCredits` 启动全量重算，回填/合并/消费链自动继承。
- `parseVolFromHtml` / `parseVolSportsFromHtml`：从志愿统计页脚本数组提取行，
  **墓碑行过滤**（容量与报名全 0 的行是已满课残影），键归一 `code_normSeq`。
- `parseTimetableCandidates`：扫一级课表脚本块的 `p_id=..;课号 … 候选：名 …
  getElementById('a{节}_{天}')`，格子 id 即真实时间正源；同课多格合并时间。
- `parseTabGrid`：分类页签 gridData 14 列解析（radio value 是课序权威源）。

### 5.5 志愿同步 `fetchVolunteer`（data.js:462）

院系定向实时拉取：

1. 池内课程按院系去重（`DEPT_CODES` 85 项映射 + `deptCodeOf` 双向 includes 兜底）。
2. 逐院系 GET 首页 → **错页校验**（返回页必须含本院系行，否则不标 done、不进 map，
   防污染自愈死循环）→ 院内分页（通常 1-3 页）。
3. **增量回调 `onData`**：每拉完一个院系立即传全量累积 map，调用方合并 `volMap` +
   `applyVolunteer` 全量重放 + 重渲——已选课卡片逐院系点亮，不等整轮扫完。
4. Ty 体育志愿独立全量拉（≤20 页）。
5. 阶段门控：仅非队列阶段执行。

三级缺行自愈（`mergeServerRows` 内调度，data.js:1830-1896）：
新院系补拉（400ms 防抖改为 60ms 立即拉）→ 院系强制重拉（`_volRetried` 每院系一次）→
逐课号定向查询 `fetchVolCourse`（BR 表单自带 `p_kch`，每课每会话一次）。
全部走"全量 volMap 重放"——`applyVolunteer` 的 else 分支无条件清空，拿增量套会
洗掉已上屏数据。

### 5.6 服务端搜索 `serverSearch` / `serverSearchStorm`（data.js:1645/1694）

- `serverSearch(opts)`：单页 kkxxSearch GET。中文参数走 `gbkPercentEncode`。
  返回 `{rows, page, totalPages, totalRows, pageKind}`，`pageKind ∈ {ok, empty, unknown}`
  （empty=结果页但 0 行；unknown=异常页带 `htmlHead` 诊断）。
- `serverSearchStorm(opts)` 风暴护栏版：
  - 精确课号（`kch` 非空且无 `kcm`）→ 只探 1 页（`forceAll` 除外——「加载全部」/
    跳转静默爬全量压过课号护栏，#32 修复）；
  - 总页数已知且 ≤25 → 全量；>25 → 只探 5 页；解析失败保守探 25 页；
  - 外校课号（非纯数字前缀）0 行 → `tabSearchByKch` 页签兜底（任选→限选→必修）；
  - 深页结果经课号前缀过滤（教务忽略筛选返回未过滤行时只收命中行）；
  - 课名 0 行且非纯数字 → 教师名通道重试一次（课名即教师名的输入习惯）；
  - 页间 30ms 错峰。**绝不做未知 25 连发**（历史实录打满代理 token）。

### 5.7 结果合并 `mergeServerRows`（data.js:1752）

搜索结果并入会话级课程池（不写 staticData 缓存）：

- `code_seq` 去重后 `allCourses.push`；
- 池内已有行：回填缺失字段（note/time/teacher/credits/department/xkTextNote），
  已有真值不动；垃圾 time（列序兜底抓到课号等）换真能解析的；
- **课号借用**（OneTHU join 语义）：已选/候补/暂存行解析不出时间 → 按课号借
  新行的 note/time；同时 `knoteRemember` 写入持久缓存；
- 暂存项同步补 note/time；
- `applyLevelMap` 补类型、`tbAttach` 补社区评价徽章（均 fail-soft）；
- 志愿统计：`volMap` 已有立刻应用到本批行；未拉院系 60ms 防抖补拉 + 三级自愈；
- 回填命中 → `invalidatePreview` + 课表重渲。

---

## 6. 阶段判定与两种数据模型

**`state.isQueuePhase`** 是全局阶段开关，由 `xkqkSearch` 首页是否含 `gridData`
判定（`fetchQueueData` 返回 `phase`）。语义：预选抽签结果出来后教务开放课余量
查询，此时进入"补选/排队"阶段。

| 维度 | 预选阶段（`isQueuePhase=false`） | 课余量阶段（`isQueuePhase=true`） |
|---|---|---|
| 概率数据源 | 志愿统计（volMap 级联） | queueDataMap（余量/排队） |
| 占用显示 | 报名人数/容量（volApplied/volCapacity） | 容量-余量=实时已选 |
| 卡片徽章 | 三行概率网格（必/限/任各 3 档 + 任选优先档） | 余X/Y · 排队N人 · 排队位次 |
| 后台同步 | `fetchVolunteer`（检查点 8/12/16/20 点） | `fetchQueueData`（`syncQueueAndVol` 同节奏） |

`occupancyOf`（probability.js:21）按阶段选占用对：预选优先志愿统计（课余量在
预选恒 0/N 造成"宽松"假象）；队列阶段优先课余量行（与"已满/余0"标签一致）。

---

## 7. 中签概率计算（probability.js）

### 7.1 志愿串格式

教务志愿统计每行三串（必修/限选/任选）+ 体育一串，形如 `(1)2,4,5`：
`(N)` 前缀 = 优先志愿人数（独立于 1/2/3 的第 0 档，任选走 `is_zyrxk=1` 通道），
逗号串 = 从高到低各志愿档人数。

`parseVolArr(s)`（probability.js:50）解析为 `[一,二,三]` 数组 + `.priority` 附加属性。
**缺位补 0 对齐右侧**：新生预选只开放第三志愿时串长 1（如 `2`），解析为
`[0,0,2]`——绝不因数量 <3 整串判 null（旧版把新生预选志愿数据全吃了）。

### 7.2 级联模型 `calcProb(course, flag, zy)`

容量按志愿优先级逐档递减分配，同档所有人平分剩余容量：

```
rem = volCapacity (或 capacity)
体育：独立级联，只看 volSports，rem 逐档减 vols[0..zy-2]
必修：flag='bx' → rem 减 bxV[0..zy-2]，返回 probResult(rem, bxV[zy-1])
      其他 flag → rem 减 bxV 全三档（必修先吃完）
限选：flag='xx' → 同上到本档；其他 → rem 减 xxV 全三档
任选：先减 priority（第 0 档），flag='rx' 再减到本档；返回本档
```

`probResult(rem, applicants)`：

- `rem <= 0` → 0% 红；
- `applicants === 0` → 100% 绿；
- 否则 `prob = min(1, rem/applicants)`；≥0.8 绿 / ≥0.5 橙 / 其余红；
- `ratioLabel = 人数/名额`（如 `2/5` = 2 人抢 5 位）；
- 无容量或无志愿数据 → `{prob:-1, label:'无数据'}` 灰。

### 7.3 展示层

- `fullProbGrid(course, bf)`：三行网格（必/限/任 × 3 档），任选行前置 `优先:N%`
  档（#39）；体育课单行。**显示侧全开**（用户十六报：志愿统计是全校公开数据），
  提交身份下拉仍按池子语义收窄。
- `currentProbLine`：卡片上"当前选法"胶囊，类型/志愿下拉 change 时由事件委托
  `syncCardProb` 原地更新（不重渲列表）。
- `volColor` / `occupancyOf`：竞争热度条（宽松/适中/激烈）。

---

## 8. 课程身份与匹配仲裁

教务系统课序号（seq）在不同页面有两套编号且前导零不一致，是本项目最大的数据
完整性威胁。防御体系：

### 8.1 键归一

`normSeq(s) = String(parseInt(s,10) || 0)`——所有跨页 map 键统一
`code + '_' + normSeq(seq)`。

### 8.2 志愿数据段匹配 `applyVolunteer`（data.js:549）

四步匹配：①原始键 ②归一化键 ③逐行归一比对 ④**多段不盲配**（该课号多个课序
且段对不上 → 宁缺毋滥返回 null，绝拿别的班的容量冒充本班；单段才允许回退）。

### 8.3 暂存/草稿身份仲裁 `courseForStage`（render.js:201）

暂存行 → 池内数据源的分层决策（#33 定案，防"王洪川 100% 错标成刘烨 9%"）：

1. 课序精确 + 教师一致（时间不矛盾）→ 直认；
2. 教师唯一直认（课序两套编号对不上是老毛病，教师才是真身份）；
3. 同师多课 → 时间消歧 → 课序仲裁 → 都分不出**诚实缺省**；
4. 无教师信息 → 裸信课序；有教师没命中 → 诚实缺省。
   （"池内唯一才兜"已废——残缺池的假唯一会关闭自动回填门槛。）

### 8.4 跳转定位仲裁 `highlightJumpTarget`（render.js:1615）

同样携带身份四参（code/seq/teacher/time），三级 findIndex 逐步放宽；未命中时：
池渲染一次性保险丝（`_jumpPoolTried` 防递归）→ 静默爬全量一轮（`_jumpAutoAll`）→
30s 意图过期。绝不高亮同名第一门充数。

### 8.5 社区评价三级匹配（reviews.js，见 §15）

### 8.6 体育课判定 `isSportsCourse`（data.js:159）

优先级：排除表（`NOT_SPORTS_NAME`：航空体育/书院专项体育/体育概论等）> 一级课表
/zyMap 属性（`attr==='体育'` / `typeLabel==='体育'` / `typeCode==='ty'`）>
体育部院系启发。分类页把非体育课误标体育时排除表仍然生效。

---

## 9. 课表解析与冲突检测

### 9.1 时间解析三级链（state.js）

**前置正源覆盖（#46）**：`fetchSelectedCourses` / `fetchCandidateCourses`
返回行先经 `overrideFromWholeTT`（data.js:1747）——以整体课表
（ztkbSearch，格子 id `a{节}_{天}` 为排课真值）覆盖 time/teacher/name。
yxSearchTab 时间列存在脏数据（同课错日/教师列缺失，教务真冲突会在选课时
被拒，故以课表为准）；暂存且未选的课不在个人课表里，map 命不中，保持原
数据链。8s 竞速超时 + 失败 60s 冷却 + 会话缓存（refreshSelected 弃缓存重拉），
任何失败回退原链。

`renderPreviewTT`（render.js:505-523）对每门课依次尝试：

1. **`parseTimeSlots(time)`**：本校格式 `3-2(全周)` = 周3-第2大节。正则
   `/(\d+)\s*[-–—]\s*(\d+)\s*\([^)]*\)/g`，结果缓存（`_slotsCache`，全校时间串
   种类有限）。
2. **manualEvent**：自定义占用直接用 day/begin/end 钟点。
3. **`clockRangesOf(note, time)`**：外校钟点解析 v2（北大/北外全格式）：支持
   周X/星期X、复合日「周二、四」「星期二/星期日」、多段「、;；」分隔、破折号
   通吃、全角括号、课级/段级单双周、周段 `(1-16周)`；返回
   `[{day, begin, end, tag}]`（分钟数）。
4. 都失败 → "时间未定"单列区（#16），并防抖触发 `backfillSelTimes` 重试。

### 9.2 时间回填 `backfillSelTimes`（state.js:172）

已选课时间解析不出（外校课典型——时间在说明列）→ 逐门 `p_kch` 精查
kkxxSearch → 课名搜兜底 → **浏览页扫描兜底**（外校课号字典序排最前，前 10 页
必含；Promise 去重防并发双扫）→ 结果经 `mergeServerRows` 回填。每课两试封顶
（`_selTried` 计数）。

### 9.3 knote 持久缓存（state.js:244-266）

凡见过能解析的时间/说明列就记入 `chrome.storage.local` 的 `knote`
（`knoteRemember`，400ms 防抖写盘）。预览 join（`previewJoinRows`）时池里没有
就用缓存兜底——外校课时间只在 kkxxSearch 说明列出现过一次也能永远用。

### 9.4 join 借用 `previewJoinRows`（state.js:272）

已选/候补/暂存行时间解析不出 → 当场按课号借池行（同课号任意班次）的
note/time 合成预览行（不改原行，返回合成副本）。每次渲染现算，不依赖回填时序。

### 9.5 冲突检测 `detectConflicts`（state.js:90）

**区间重叠制**（OneTHU 同款）：课程大节 → 钟点区间（`SLOT_RANGE`），自定义占用
直接钟点区间；`begin < s.end && s.begin < end` 判重叠——跨边界部分重叠
（8:00-9:35 vs 9:00-10:30）也能测出（旧版按"同日同大节"精确匹配全漏）。

`findPreviewConflicts`（state.js:387）是卡片级版本：查单课与当前预览课表的
冲突，走 `previewSlotIndex`（槽位索引，引用+长度双重失效缓存）。

### 9.6 动态时间轴渲染（render.js `renderPreviewTT`）

- 轴：08:00-21:45 起步（`PV_AXIS_*`），0.72px/分钟（`PV_PX_PER_MIN`），随课块
  伸缩（30 分钟对齐）——自由时间轴，占用可越出默认轴。
- **同日重叠分道**（簇算法，render.js:537-563）：簇 = 首尾相接/重叠的块序列，
  簇内独立分道（lane/lanes），孤立块满宽。绝不用全日总道数劈半天。
- 课块着色：候选=橙（排队位次）、已选(队列阶段)=绿、自定义=紫、暂存/草稿=
  概率色、无概率色按课名稳定取色（`pvColorOf` 哈希调色板）。暂存/草稿快照行
  不带 `selected/isCandidate`，stage/draft 预览经 `stageStatusOf` 回池仲裁——
  命中已选=绿「已选」、命中候补=橙「排队第X/Y人」，优先于概率/余量标签。余量
  标签同 `stageProbHtml` 三层兜底：queueDataMap 快照键 → 池行键（courseForStage）
  → 池行合成（capacity/remaining）；草稿专属课经 `backfillStageRows` 补拉落池后
  仅由池行合成兜底点亮预览标签（草稿列表行不显示课余量，kyl 查询集不并入草稿课号）。
- 课块可直接操作：✕ 移除（已选退选走警告弹窗 / 暂存移除 / 草稿删除）、点击
  跳转定位。

---

## 10. 渲染层

### 10.1 渐进渲染 + 事件委托

- `renderCourses(list)`：重置 `state.renderList/renderCursor/renderCtx`，插入哨兵
  节点，`IntersectionObserver`（rootMargin 800px）驱动 `renderMoreCourses` 每批
  渲染 `RENDER_CHUNK=80` 张卡片。
- `bindCardDelegation(el)`：容器级 click/change 两个监听器（`el.dataset.nxDelegated`
  防重复绑定），替代每按钮闭包。click 分发到 `.nx-detail-btn`（简介）、
  `.nx-tb-badge`（社区点评）、`.nx-select-btn`（选课）、`.nx-drop-btn`（退课，
  先过 `confirmDrop` 玻璃警告）、`.nx-add-stage`（暂存）、`.nx-vol-btn`（志愿
  ▲▼）、`.nx-add-draft`（加入当前草稿）；change 分发到类型/志愿下拉 →
  `syncCardProb` 原地更新概率胶囊。
- 退课成功**本地摘牌**（#36-5）：不等 refreshSelected 网络往返，立即清
  selected/isCandidate/zy 并重渲，防二连击。

### 10.2 渲染上下文

`renderCtx = {candMap, stageSet}`：候补/暂存索引在渲染前一次建好，卡片 HTML
生成时 O(1) 查。

### 10.3 其余渲染函数

| 函数 | 目标容器 | 说明 |
|---|---|---|
| `renderPreviewTT` | `#nextthuxk-preview-tt` | 时间轴课表（§9.6） |
| `renderQueueSection` | `#nextthuxk-queue-list` | 右栏候选队列（排队位次 + 退队） |
| `renderStageCart` | `#nextthuxk-stage-list` | 暂存区（每课独立调类型/志愿 + 已选/排队徽章（`stageStatusOf` 回池仲裁）+ 概率网格 + 冲突汇总） |
| `renderDrafts` | `#nextthuxk-drafts` | 草稿卡列表（展开/预览载入/提交/导出/删除；展开行含已选/排队徽章）；`refreshSelected` 选退课后重渲同步徽章 |
| `renderPlan` / `renderPlanView` | `#nextthuxk-plan` / `#nextthuxk-list` | 右栏方案进度卡 / 左栏方案分组视图 |
| `showCourseModal` | `#nextthuxk-modal` | 课程简介弹窗（fetchCourseDetail） |
| `renderListFooter` | `#nextthuxk-list` 尾部 | 分页条 + 数据不完整提示 + 加载全部 |

### 10.4 预览缓存失效体系

三处版本号缓存，数据变化时显式失效：

| 缓存 | 失效键 | 失效方式 |
|---|---|---|
| `_selCache`（已选预览行） | `selVersion + 候选数 + poolVersion` | 版本号比对自动失效 |
| `_pvIdx`（槽位索引） | 数组引用 + 长度签名 | `invalidatePreview()` 手动清 |
| `courseMap`（code_seq 索引） | — | `rebuildCourseMap()` 手动重建 |

---

## 11. 搜索管线（随时查询模式）

2.x 核心架构：课程列表**绝不整库预爬**，全部服务端随时查。管线（render.js:1160+）：

```
用户输入/筛选变化
→ filterCourses()
   ├─ chip='plan' → renderPlanView（本地）
   ├─ 本地 chip（selected/queue/required/elective/sports）→ 池内过滤
   └─ 其余：
      ├─ serverSigOf() 计算服务端条件指纹（关键词+SEM+页码+7个服务端筛选）
      │   注意：chip / conflict / credits / reviews / sort / xknote 不入指纹
      │       （纯本地细化，切换即时生效不重查）
      ├─ 指纹变了 → _searchRows=null, _uiPage=1 → scheduleServerSearch()
      │             列表显示"正在查询教务…"后 return
      └─ 未变 → list = _searchRows → 本地细化过滤（见下）
         ├─ 关键词（name/code/teacher 模糊）
         ├─ chip：available / required / elective / sports
         ├─ activeGroup（培养方案组）
         ├─ 学分 / 周次-大节（预编译正则）/ 冲突 / 通识组 / 特色 / 年级
         ├─ 本研余量 / 选课文字说明 / 社区评价（_tbRef）
         └─ 排序（社区评分/点评数，无点评恒排末尾；复制数组不动池序）
      → 搜索模式本地分页（PAGE_SIZE=20 页切片）
      → renderCourses(show) + renderListFooter()
```

### 11.1 查询执行 `runServerSearch`（render.js:1534）

- `_ssBusy` 互斥；跑动中条件再变 → `_ssPending` 排队，收敛后自动补跑
  （最多 4 轮 guard）。
- 浏览模式（全条件空）→ 单页 `serverSearch(page=_browsePage)`；查询模式 →
  `serverSearchStorm`。
- 课号 0 行且池里有该课 → 自动换课名重搜（外校课号索引缺失恢复）。
- 结果行打 `selected/isCandidate` 标记后写 `_searchRows`，并 `mergeServerRows`
  并入会话池（暂存/简介/选课按钮即刻可用）。
- **全量降级守卫**：同查询全量已在手（`_searchRowsFull` + 签名一致）→ 浅层结果
  不回写（防把 112 行全量降级回 20 行，#33）。
- `_searchIncomplete`：已加载 < 服务端总数 → 分页条出「加载当前关键词全部」。

### 11.2 课号路由 `isCodeLike`（render.js:1511）

≥5 位、无中文、含数字、字母数字连字符 → 课号（`kch`，截 `-` 前段）；否则课名
（`kcm`）。`buildSearchOpts` 把 UI 条件翻译为查询参数。

### 11.3 补齐

- `loadSearchPageTo(target)`：跳页补载（目标页 > 已装载页 → 逐页补拉）。
- `loadAllSearch()`：forceAll 全量补齐，完成后校验仍缺则保留提示可重试；
  `_searchRowsFullTag` 记录全量签名。

### 11.4 概率自动回填 `backfillStageProbs`（render.js:779）

`renderStageCart`/`renderDrafts` 尾部调度（700ms 防抖）：池里查不到或志愿统计
缺失的已选/暂存/草稿课 → 按课号 `serverSearchStorm({kch, forceAll:true})` 静默
爬全量合并进池；每课号每会话 2 次封顶；与查询管线互斥（`_ssBusy/_loadingAll`
跑动中延后，防把全量盖回去）。概率不再要用户点一下才出现。

---

## 12. 暂存 / 草稿 / 提交流程

### 12.1 数据模型

- **stageCart**（暂存区，storage 持久）：课程快照数组，每项含 `flag`（提交类型）/
  `zy`（志愿号）/ `baseFlag`（课程本身类型，迁移用）/ `note`（外校时间）。
  launch / refreshSelected 时 `syncStageWithWholeTT`（state.js:613）把其中已是
  已选/候补的课刷新为整体课表正源（课表外的暂存项保持原快照）。
- **savedDrafts**（草稿，storage 持久，最多 5 份）：`{id, name, courses, createdAt}`，
  courses 与 stageCart 同构。满 5 份时 `askReplaceDraft` prompt 选替换。

### 12.2 操作流

```
卡片「暂存」→ addToStage（knoteRemember + 快照入 stageCart + 持久化）
已选课 → saveSelectedAsDraft（typeCode → flag 映射）
暂存区 → saveDraft（清空 stageCart 转草稿）
草稿「预览 & 修改」→ 载入暂存区（替换确认）
草稿「导出」→ exportDraft（JSON 复制，clipboard 降级链：writeText →
              execCommand → 弹 textarea 手动 Ctrl+C，任何路径有反馈）
导入 → importToStage（JSON 解析 + 去重并入 + baseFlag 迁移）
草稿「提交选课」→ promoteDraft（见下）
```

### 12.3 差量提交 `promoteDraft`（state.js:742）

不盲目"全退全选"，而是**差量比对**（key = `code_normSeq`）：

1. 拉取当前已选 + 候补；
2. 分三类：`toDropSel`（已选但草稿没有）、`toDropQueue`（候补但草稿没有）、
   `toSubmit`（草稿有但当前没有）；其余保留不动；
3. 明细确认弹窗（列出三分类全清单）+ 二次确认；
4. 顺序执行：退选（每门间隔 1s）→ 退队 → 选入（每门间隔 2s，避开验证码），
   toast 显示进度；
5. `refreshSelected` 全量刷新 + 预览重渲；中途出错不回滚（明确告知用户）。

### 12.4 志愿上限校验 `canAdjustZy`（state.js:832）

`ZY_LIMITS`：必/限/任 1 志愿最多 1 门、2 志愿最多 2 门、3 志愿不限；体育
1/2 志愿各最多 1 门。调整前按 `zyTypeOf` 分类统计已选计数校验。

---

## 13. 选课 / 退选 / 志愿调整 API

### 13.1 通用表单提交 `fetchFormSubmit`（data.js:573）

所有写操作走同一管道（1:1 模拟教务 UI 表单）：

```
GET 搜索页提取 token
→ POST xkBks.vxkBksXkbBs.do（token + 业务字段）
→ 响应检测（GBK 解码）：
   ├─ accessDenied → 会话失效
   ├─ 「加入队列成功」/「选课成功」→ ok:true submitted:true
   ├─ 拒绝字典 REJECT_RE（时间冲突/先修/余量不足/已满/验证码/上限…20+ 模式）
   │   → ok:false + alert 文案或正文摘录（旧版未知响应一律假成功，此处修复）
   ├─ 「是否排队」confirm → 提取响应页新 token（原 token 一次性已消耗！）
   │   → 二次 POST m=saveBksKcDl → 再检测
   └─ 未知 → {ok:false, unknown:true} 交调用方轮询确认
```

### 13.2 业务封装

- `submitCourse(code, seq, zy, flag)`：按 flag 路由搜索页（bxSearch 等）与
  id/zy 字段名（p_bxk_id/p_bxk_xkzy 等，任选附 `is_zyrxk=1`）。提交后
  `pollUntil`（700ms×3）轮询已选/候补列表确认结果——**未知响应也轮询确认**，
  确认命中才算成功。
- `dropCourse(code, seq)`：按 `candidateCourses` 路由——候补课走 `m=dlDelete`
  （token 取自 dlSearchTab 页），已选课走 `m=deleteYxk`（token 取自 yxSearchTab）。
  轮询消失确认。
- `changeVolunteer(code, seq, targetZy)`：`m=changeZY`。
- `pollUntil(fn, delay, tries)`：轮询替代固定 sleep（平均省 1s+）。

### 13.3 已选课解析 `fetchSelectedCourses`（data.js:1060)

- `yxSearchTab` 页：先从 HTML 脚本数组提取 `zyMap`（课号_课序 → 志愿号/类型/
  是否体育），再解析 `tr.trr2` 行；**列序自适应**（2026-2027-1 起课号独立成列，
  取第一个非纯数字候选格，新旧列序通吃）。
- 空结果兜底：`fallbackSelectedFromLevelTable`（一级课表重建 code+seq+类型，
  志愿走 zyCache/手填）。

---

## 14. AI 层（ai.js）

兼容 OpenAI 格式的 `/chat/completions`。配置（api/model/token/pref）存
storage `config`，launch 时回填。

- `aiCourseJson(c)`：课程 → 统一精简 JSON（含志愿统计 vol、余量、候补/暂存标记、
  社区评分 reviewAvg/reviewCount/latestReview、外校说明 note）——aiSearch 与
  callAI 共用的"单一真相"。
- `aiOccupied(previewCourses)`：预览占用归一（大节 + 外校钟点 + 自定义占用），
  prompt 里告知 AI 哪些时段真没了。
- `aiSearch`：在**当前筛选结果**（`_searchRows` 优先，退池内）中推荐，返回
  JSON `{recommendations:[{code,seq,name,reason,conflict,conflictWith}],summary}`；
  冲突课显式标注；每条带「暂存」按钮。
- `callAI`：智能排课。候选 = 当前搜索结果 ∪ 池内必修/体育；注入已选课表/占用
  时段/全部草稿/用户偏好/年级约束（体育(1)(2)(3) 年级段规则）；返回
  `{courses,total_credits,summary,suggestions}`，结果直接载入暂存区并保存为
  「AI推荐」草稿；`detectConflicts` 复核冲突。
- 两处 prompt 都注入 THU选课社区评价参考段（CC BY-NC 4.0 署名要求）。
- temperature 0.3；响应剥 markdown 围栏后 JSON.parse。

---

## 15. THU选课社区评价层（reviews.js）

IIFE 封装，数据源 `https://thubook.help/data/`（公开静态 JSON，CORS *）。
**设计原则**：只缓存精简索引（count/avg/sqid），正文实时拉取不囤积；全程
fail-soft。

### 15.1 索引 SWR

`tbEnsureIndex`（reviews.js:102）：

- storage 读缓存（`tbookIdx`，版本 `IDX_VER=1`）→ 立即 `buildMaps` 建三索引
  （`bySqid` / `byNameT` / `byName`）；
- 缓存超 24h（`IDX_TTL`）→ 后台刷新；新鲜 → 也静默预热到最新（失败明天再说）；
- 单飞行 promise（`S.loadingPromise`）防并发；
- 刷新后 `reattachAll` 重挂徽章。

### 15.2 三级匹配器 `tbMatch`（reviews.js:146）

- **T1**：课名（NFKC 折叠全角/去空白/小写）+ 教师 token 集（排序连接）双精确；
- **T2**：同名桶内教师 token 相交评分（`10+交集数`，单方无教师信息=中分 5，
  热门度微加权 `min(count,10)×0.01`）取最优；
- **T3**：命名漂移兜底——去 `(英)/荣誉/尾缀序号` 再试一次，**必须过教师核对**
  （否则英文班会错吸中文班评价）；
- miss 记入 `S.stats`。实测 1345 门有点评课程 100% 命中。

### 15.3 正文与 UI

- `tbFetchReviews(sqid)`：正文实时拉 + 10min 内存缓存（`DETAIL_TTL`），跟随
  `next` 分页最多 5 跳，按时间倒序。
- `showReviewsModal`：复用全局玻璃 Modal；头部评分 + 星星 + CC BY-NC 署名声明 +
  课程页/写点评深链；无匹配给搜索兜底链接。
- 联想词（`suggestUpdate`）：本地池打分（前缀 100 / 包含 80 / 课号 70 / 教师
  60-50 / 课号包含 40）+ 社区评分热度微加权（≤5 分）+ 余量加 2；键盘导航
  （`suggestKey` 上下/回车/ESC）；pointerdown preventDefault 保焦。
- 徽章 `tbBadgeHtml`（render.js）：≥4.5 绿 / ≥4 靛 / ≥3 琥珀 / <3 红，
  点击开弹窗。

---

## 16. 更新检查（update.js）

- `cmpVer(a,b)`：三段版本号比较。
- `checkUpdate`：危险版本（`DANGEROUS_VERS`）直接红横幅；否则 GitHub Releases
  latest API，30 分钟节流（`lastUpdateCheck`），发现新版显示玻璃横幅（提醒先移除
  旧版再装防实例冲突）；`setInterval` 30 分钟周期复查。
- `showUpdateBanner` / `showDangerBanner`：`db.prepend` 注入，✕ 关闭。
- 版本单源：`CUR_VER` 读 `runtime.getManifest().version`（v2.0.1 双版本源事故
  后修复）。`BUILD = 'rt-park1'` 构建标记显示在顶栏（防"页面刷新了但扩展没刷新"）。

---

## 17. 容错设计汇总

| 机制 | 位置 | 说明 |
|---|---|---|
| fail-soft | reviews.js 全部、tbAttach 调用处 | 评价层任一环节失败不影响选课主流程 |
| 静默降级链 | fetchCandidateCourses | dlSearch 失败/0 行 → kbSearch 课表兜底 → 安静空着不报错 |
| 兜底重建 | fetchSelectedCourses | yxSearchTab 空 → 一级课表重建已选清单 |
| 三级搜索兜底 | backfillSelTimes | 课号 → 课名 → 浏览页扫描（共享单次扫描） |
| 熔断 | pagedFetch 补抓 / fetchQueueData 批量 | 连续 5 页 / 3 批失败即停，防 session 失效时轰炸服务器 |
| 票据自愈 | fetchPage + reenterZhjwxk | WebVPN 壳页自动换票（60s 冷却），SSO 页明确指引重登 |
| 拒绝字典 | fetchFormSubmit | 未知响应绝不假成功；unknown 轮询确认 |
| 诚实缺省 | courseForStage / applyVolunteer | 分不出身份就不显示概率，绝不冒认 |
| 竞态守卫 | launch SEM0 / _ssBusy / 全量降级守卫 | 学期切换弃写缓存；查询互斥排队；全量不被浅层覆盖 |
| 两试封顶 | _selTried / _probBfTried / _volRetried | 所有自动重试都有会话级次数上限，不刷屏不打爆服务端 |
| 错位页校验 | fetchVolunteer | 返回页无本院系行不标 done，防污染自愈死循环 |
| 意图过期 | _jumpAt 30s | 跳转高亮意图防陈旧串场 |

---

*文档生成于 v2.1.1（构建 rt-park1）。行号引用以当前源码为准。*
