# NextTHUxk 变量说明手册

> 本文档逐项说明项目中的常量、全局状态、核心数据结构与持久化存储键。
> 架构与代码逻辑见 [ARCHITECTURE.md](./ARCHITECTURE.md)。行号以 v2.1.1 源码为准。

---

## 目录

1. [NX 命名空间常量](#1-nx-命名空间常量)
2. [NX.state 全局状态](#2-nxstate-全局状态)
3. [课程对象（核心数据结构）](#3-课程对象核心数据结构)
4. [志愿数据结构](#4-志愿数据结构)
5. [课余量与队列数据](#5-课余量与队列数据)
6. [暂存项与草稿对象](#6-暂存项与草稿对象)
7. [课表时间结构](#7-课表时间结构)
8. [评价层变量（reviews.js）](#8-评价层变量reviewsjs)
9. [编码层变量（gbk.js）](#9-编码层变量gbkjs)
10. [存储键清单（chrome.storage.local）](#10-存储键清单chromestoragelocal)
11. [Shadow DOM 元素 ID 索引](#11-shadow-dom-元素-id-索引)
12. [模块级杂项变量](#12-模块级杂项变量)

---

## 1. NX 命名空间常量

全部定义于 `src/config.js`（另注明者除外），运行期只读。

### 1.1 基础标识（config.js:10-23）

| 变量 | 值 | 说明 |
|---|---|---|
| `NX` | `window.NX = {}` | 全局命名空间单例，所有模块向其挂载函数/常量（config.js:5） |
| `NX.browser` | `browser \|\| chrome` | 跨浏览器 API 入口（config.js:7） |
| `NX.TAG` | `'[NextTHUxk]'` | 控制台日志前缀 |
| `NX.SP` | `'nextthuxk_'` | 存储键前缀，`store.get/set` 自动拼接 |
| `NX.DATA_VER` | `6` | staticData 缓存结构版本；不符即清空缓存并重置年级（content.js:276） |
| `NX.CUR_VER` | 读 manifest | 当前版本号，单源取自 `runtime.getManifest().version`（防双版本源自报错版） |
| `NX.BUILD` | `'rt-park1'` | 构建标记，显示于顶栏 logo 旁（防"页面刷新了但扩展没刷新"的旧构建疑案） |
| `NX.DANGEROUS_VERS` | `['1.0.1','1.0.2','1.0.3','1.1.2','1.2.0']` | 存在严重错误必须升级的历史版本，命中即红横幅 |
| `NX.ZY_LIMITS` | 见下 | 志愿名额上限规则表 |

`NX.ZY_LIMITS` 结构：`{ flag: [[志愿号, 上限], ...] }`

```js
bx: [[1,1],[2,2],[3,Infinity]]   // 必修：1志愿最多1门，2志愿最多2门，3志愿不限
xx: [[1,1],[2,2],[3,Infinity]]   // 限选：同必修
rx: [[1,1],[2,2],[3,Infinity]]   // 任选：同必修
ty: [[1,1],[2,1],[3,Infinity]]   // 体育：1志愿1门，2志愿1门
```

### 1.2 类型体系（data.js:142-190）

flag 四值：`bx` 必修 / `xx` 限选 / `rx` 任选 / `ty` 体育。

| 变量 | 说明 |
|---|---|
| `NX.NOT_SPORTS_NAME`（data.js:158） | 正则 `/航空体育\|书院专项体育\|体育(概论\|管理\|课程与教学论\|科技前沿)/`。**非全校统一体育课排除表**——这些课混在体育部列表但没人走体育通道，判定优先级最高 |
| `NX.ORIGIN_COLORS`（state.js:85） | `{北大:'#c0392b', 北大研:'#c0392b', 北外:'#1f4e79'}`，外校课标签/课块着色 |
| `NX.DEPT_CODES`（data.js:338-424） | 85 项院系名 → 数字码映射表（如 `'计算机系': '024'`），志愿统计按院系定向拉取用 |
| `NX._GBK_URL_RE`（config.js:83） | `/zhjw\|xkBks\|jhBks\|vjsKcbBs/`，编码探测兜底时判定"教务默认 GBK"的 URL 特征 |

### 1.3 渲染常量（render.js）

| 变量 | 值 | 说明 |
|---|---|---|
| `PV_BEGIN` / `PV_END`（render.js:7-8） | 数组 `['', '08:00', …, '21:00']` / `['', '08:45', …, '21:45']` | 清华第 1-14 小节的标准上下课钟点 |
| `NX.SLOT_RANGE`（render.js:11） | 6 元素数组 | 六个大节（1-2 / 3-4 / 5-6 / 7-8 / 9-10 / 11-12 节）对应的 `[起始分钟, 结束分钟]` 区间 |
| `NX.PV_PX_PER_MIN`（render.js:19） | `0.72` | 时间轴每分钟像素数 |
| `NX.PV_AXIS_BEGIN`（render.js:20） | `480`（08:00） | 时间轴默认起点（分钟） |
| `NX.PV_AXIS_END`（render.js:21） | `PV_END[14]` = 1305（21:45） | 时间轴默认终点（分钟）；实际轴随课块伸缩 |
| `PV_PALETTE`（render.js:23） | 10 色数组 | 课块无概率色时的稳定取色调色板 |
| `NX.RENDER_CHUNK`（render.js:57） | `80` | 渐进渲染每批卡片数 |
| `NX.PAGE_SIZE`（render.js:1321） | `20` | 搜索/浏览模式每页条数（教务同款） |

### 1.4 同步检查点（content.js:517）

| 变量 | 值 | 说明 |
|---|---|---|
| `NX.VOL_CHECKPOINTS` | `[8, 12, 16, 20]` | 教务志愿数据每日刷新时刻（时）。`startVolAutoSync` 据此调度 `syncQueueAndVol`，`nextVolCheckpoint` 计算下次同步时间显示在顶栏 |

### 1.5 评价层常量（reviews.js:15-22，IIFE 内）

| 变量 | 值 | 说明 |
|---|---|---|
| `TAG` | `'[NextTHUxk][TB]'` | 评价层日志前缀 |
| `TB_PAGE` | `https://thubook.help/thucourse/` | 社区页面根 |
| `TB_DATA` | `https://thubook.help/data/` | 社区静态数据根 |
| `IDX_KEY` / `TS_KEY` | `'tbookIdx'` / `'tbookIdxTs'` | 索引缓存键 / 索引时间戳键 |
| `IDX_VER` | `1` | 索引结构版本 |
| `IDX_TTL` | `24h` | 索引过期阈值，超时后台静默刷新 |
| `DETAIL_TTL` | `10min` | 点评正文内存缓存时长 |
| `SG_MAX` | `8` | 联想词下拉最多条数 |

---

## 2. NX.state 全局状态

定义于 `src/config.js:26-48`，`content.js` 启动时填充。**唯一的可变全局**——
运行期各模块直接读写其字段。

### 2.1 站点与学期（content.js:22-31 初始化）

| 字段 | 类型 | 说明 |
|---|---|---|
| `SEM` | string | 当前学期，如 `'2026-2027-1'`。优先取 URL `p_xnxq` 参数 → storage → prompt 手输 |
| `GRADE` | number | 年级 1-4（0=未设置）。仅影响 AI 对体育课年级段的推荐 |
| `BASE` | string | 教务请求基址。普通站点 = `location.origin`；WebVPN = origin + `/{protocol}/{encoded-host}` 编码站点前缀（缺它则全部 404） |
| `isZhjwxk` | bool | 主选课站 zhjwxk.cic.tsinghua.edu.cn |
| `isZhjw` | bool | 成绩单站 zhjw.cic.tsinghua.edu.cn（培养方案走不同解析器） |
| `isWebvpn` | bool | WebVPN 环境（content.js 动态挂载；`ensureSiteIdentity` 会 AES 解密路径主机段复核上两者） |

### 2.2 数据池

| 字段 | 类型 | 说明 |
|---|---|---|
| `allCourses` | Course[] | **会话级课程池**：启动 = 已选 + 候补；之后搜索/回填结果经 `mergeServerRows` 持续并入。渲染、选课、冲突检测的统一数据源 |
| `planData` | PlanItem[] | 培养方案课程列表（storage staticData 缓存） |
| `candidateCourses` | Course[] | 候补队列（dlSearch / kbSearch 兜底解析，`isCandidate:true`） |
| `levelMap` | Object | 一级课表 + 分类页属性合并的类型索引：`code_normSeq → {typeCode, typeLabel, attr}`。kkxxSearch 行没有类型列，全靠它回填 |
| `volMap` | Object | 志愿统计累积 map：`code_normSeq → volRow`（见 §4）。全局持久于会话，搜索/跳转新行可取 |
| `queueDataMap` | Object | 课余量/排队 map：`code_normSeq → {qCapacity, qRemaining, qQueue}` |
| `knote` | Object | 课表时间持久缓存（内存镜像，storage `knote`）：`code_seq → {note, time}` |
| `courseMap` | Map | `code_seq → course` 索引（`rebuildCourseMap` 重建），`getCourse` O(1) 查询 |

### 2.3 阶段与预览

| 字段 | 类型 | 说明 |
|---|---|---|
| `isQueuePhase` | bool | **阶段开关**：true=补选/排队阶段（课余量模型），false=预选/志愿期（概率级联模型）。由 xkqkSearch 首页 gridData 有无判定 |
| `previewMode` | string | `'selected' \| 'stage' \| 'draft'`，课表预览当前视图 |
| `previewDraftIdx` | number | 草稿预览索引（-1=无） |
| `expandedDraft` | number | 右栏展开的草稿索引（-1=无） |
| `activeGroup` | string \| null | 培养方案组过滤（`filterByGroup` 设置） |
| `manualEvents` | ManualEvent[] | 自定义时间占用（storage 持久），参与全部冲突检测 |

### 2.4 暂存与草稿

| 字段 | 类型 | 说明 |
|---|---|---|
| `stageCart` | StageItem[] | 暂存区课程快照（storage `stageCart`） |
| `savedDrafts` | Draft[] | 已保存草稿，最多 5 份（storage `drafts`） |

### 2.5 UI 引用

| 字段 | 类型 | 说明 |
|---|---|---|
| `host` | HTMLElement | `#nextthuxk-host` 宿主节点 |
| `shadow` | ShadowRoot | open 模式 Shadow Root |
| `$` | Function | `id => shadow.getElementById(id)`，全项目元素查询入口 |
| `updateTimer` | number | 更新检查 setInterval 句柄 |

### 2.6 渲染管线状态（render.js 动态挂载）

| 字段 | 说明 |
|---|---|
| `renderList` | 当前渲染的全量列表（filterCourses 产出） |
| `renderCursor` | 渐进渲染已渲染到的索引 |
| `renderSentinel` | 哨兵节点（IntersectionObserver 观测点） |
| `renderObserver` | IntersectionObserver 实例 |
| `renderCtx` | `{candMap: Map, stageSet: Set}` 渲染上下文（候补/暂存索引） |

### 2.7 搜索管线状态（render.js 动态挂载）

| 字段 | 说明 |
|---|---|
| `_searchRows` | 当前查询的服务端结果行（本地筛选/分页的基底） |
| `_searchRowsFull` / `_searchRowsFullTag` | 全量标记 + 全量时的条件指纹——同指纹浅层搜索不得降级回写（#33） |
| `_searchTotalPages` / `_searchTotalRows` | 服务端总页数 / 总条数（分页条真值） |
| `_searchIncomplete` | 已加载 < 服务端总数 → 显示「加载当前关键词全部」 |
| `_searchError` | 查询异常文案（unknown 页 / 网络失败，显式上屏 + 重试按钮） |
| `_serverSig` | 服务端条件指纹（`serverSigOf()`：关键词+SEM+页码+7 个服务端筛选；chip/冲突/学分等本地细化不入指纹） |
| `_uiPage` | 搜索模式本地页码 |
| `_browsePage` / `_browseHasMore` | 浏览模式页码 / 是否还有下一页 |
| `_ssBusy` / `_ssPending` | 查询互斥锁 / 跑动中条件再变的排队标志 |
| `_loadingAll` / `_loadAllPending` / `_searchDeferred` | 「加载全部」进行中 / 排队 / 查询延后标志 |

### 2.8 跳转定位状态（render.js:680-686 写入）

| 字段 | 说明 |
|---|---|
| `_jumpCode` / `_jumpSeq` | 跳转目标课号 / 课序 |
| `_jumpTeacher` / `_jumpTime` | 跳转身份消歧用教师 / 时间（教师仲裁：同课号同课序多教师时定位到对的人） |
| `_jumpPoolTried` | 池渲染一次性保险丝（防递归） |
| `_jumpAutoAll` | 允许静默爬全量补齐一轮（#33） |
| `_jumpCrawling` / `_jumpAt` | 爬取中标志 / 意图时间戳（30s 过期防陈旧串场） |

### 2.9 其余运行时状态（动态挂载）

| 字段 | 定义处 | 说明 |
|---|---|---|
| `launching` | content.js:241 | launch 并发锁（防双击双抓） |
| `fetchWarn` | data.js:301 等 | 数据不完整提示文案，launch 结束时 toast 一次性展示 |
| `selVersion` | state.js 多处递增 | 已选集版本号（`_selCache` 失效键组成部分） |
| `poolVersion` | data.js:1801 递增 | 池内容版本号（join 预览缓存失效键组成部分） |
| `_volDepts` | data.js:468 | 本会话已拉院系表 `{院系码: 时间戳}`（force 重拉绕过） |
| `_volRetried` | data.js:1837 | 缺行自愈重试表 `{院系码\|'k:'+课号: 1}`（每键每会话一次） |
| `_volDebounce` | data.js:1846 | 志愿补拉防抖句柄 |
| `_volSyncStarted` / `_volSyncT` / `_nextVolSyncAt` / `_lastVolSyncAt` | content.js:547 | 检查点定时同步：启动标志 / setTimeout 句柄 / 下次同步时间 / 上次同步时间 |
| `_selVolTried` | content.js:366 | 已选课志愿定向补拉已试课号表 |
| `_selTried` | state.js:175 | 已选时间回填计数 `code_seq → 次数`（两试封顶，「重试解析」按钮清零重试） |
| `_selBfLogged` | state.js:183 | 无可查日志只打一次标志 |
| `_bfStatus` | state.js:191 | 回填状态行 `code → '查询中' \| '✓已上轴' \| '×…'`（时间未定区显示） |
| `_bfScanP` | state.js:211 | 浏览页扫描共享 promise（防并发双扫，「重试」时作废重扫） |
| `_undetBfT` | render.js:531 | 时间未定触发回填的防抖句柄 |
| `_probBfTimer` / `_probBfTried` / `_probBfDeferred` | render.js:775-788 | 概率回填：防抖句柄 / 每课号两试计数 / 管线互斥时延后计数 |
| `_reenterAt` | data.js:1611 | WebVPN 换票 60s 冷却时间戳 |
| `_wholeTTP` / `_wholeTTSem` / `_wholeTTFail` | data.js:1711-1745 | 整体课表（ztkbSearch）会话缓存 promise / 缓存学期键 / 失败 60s 冷却时间戳（refreshSelected 弃缓存重拉正源） |
| `_siteP` | content.js:196 | `ensureSiteIdentity` 单飞行 promise |
| `_rating*` | data.js 注释块 | 官方教评（#31）已冻结，全部注释保留 |

---

## 3. 课程对象（核心数据结构）

课程对象贯穿全项目，多个解析器产出、多处合并回填。全集字段：

### 3.1 目录/搜索行（`parseCatalog`，data.js:72-97）

| 字段 | 类型 | 说明 |
|---|---|---|
| `code` | string | 课号，8 位数字（本校）或 `PK`/`GPK`/`BW` 前缀（外校） |
| `seq` | string | 课序号（同一课号多个班次）。**注意两套编号与前导零不一致，跨页匹配必须 `normSeq` 归一** |
| `name` | string | 课程名 |
| `teacher` | string | 教师名（可能多教师逗号分隔） |
| `teacherId` | string | 教师 ID（从教师链接 `p_jsh=` 提取，简介弹窗 `fetchCourseDetail` 用） |
| `credits` | number | 学分。**规则**：本校课（课号纯数字）= 课号最后一位（`NX.lastDigitCredits`，config.js）；外校课（课号含字母）= 学分列原解析值。暂存/草稿旧快照启动时全量重算（`NX.migrateStageCredits`） |
| `department` | string | 开课院系名 |
| `time` | string | 本校时间串，如 `'3-2(全周)'`（周3-第2大节）；外校课此字段通常为空 |
| `note` / `xkTextNote` | string | 说明列（kkxxSearch 第 12 列）——**外校课真实上课时间的载体**（`clockRangesOf` 解析） |
| `capacity` / `remaining` | number | 本科容量 / 余量 |
| `gradCapacity` / `gradRemaining` | number | 研究生容量 / 余量 |
| `available` | bool | `remaining > 0`（有余量） |
| `attr` | string | 课程属性：`'必修' \| '限选' \| '任选' \| '体育' \| ''`（kkxxSearch 行无类型列，靠 levelMap/分类页/培养方案回填） |
| `group` | string | 课组（取开课单位列） |
| `detailUrl` | string | 课程简介页相对链接 |
| `courseFeature` | string | 课程特色（专题研讨课/全外文授课…筛选用） |
| `grade` | string | 年级限制串 |
| `tongshiGroup` | string | 通识课组（人文/社科/艺术/科学） |
| `queue` | string | 遗留字段（恒 `''`） |

### 3.2 状态标记（多来源写入）

| 字段 | 写入处 | 说明 |
|---|---|---|
| `selected` | fetchSelectedCourses / resolveCourseZy / 搜索结果标记 | 是否已选 |
| `isCandidate` | fetchCandidateCourses / syncQueueAndVol | 是否候补队列课 |
| `zy` | number | 志愿号 1-3（0=未知/未选） |
| `typeCode` | string | 教务类型码：`'006'` 必修 / `'008'` 限选 / `'007'` 任选 / `'ty'` 体育 |
| `typeLabel` | string | 类型中文名（必修/限选/任选/体育） |
| `fromLevelTable` | bool | 已选兜底来源标记（一级课表重建，缺名称/时间/学分） |
| `partial` | bool | 分类页签行标记——元数据未由全量目录补全（余量未知≠已满） |

### 3.3 志愿统计回填（`applyVolunteer`，data.js:560-563）

| 字段 | 说明 |
|---|---|
| `volRequired` / `volElective` / `volOptional` | 必修/限选/任选各志愿报名串，原始格式 `'(1)2,4,5'`（见 §4.2） |
| `volSports` | 体育志愿串 |
| `volCapacity` / `volApplied` | 志愿统计口径的容量 / 已报人数（概率计算数据源；缺行=无数据，不回退目录容量） |

### 3.4 队列阶段回填（launch / syncQueueAndVol）

`available` / `remaining` / `capacity` 会被 queueDataMap 覆写（qRemaining>0 才覆写 remaining）。

### 3.5 候补行专有（`fetchCandidateCourses`，data.js:1295-1302）

| 字段 | 说明 |
|---|---|
| `queueTotal` | 该课候补总人数 |
| `myPos` | 本人排队位次（kbSearch 兜底候选无此数据，显示"候选中"） |

### 3.6 评价层附加（`tbAttach`）

| 字段 | 说明 |
|---|---|
| `_tbRef` | 匹配到的社区索引条目 `{kcm, jsm, kkdw, sqid, tid, count, avg, nt}`（无匹配时删除） |
| `_tbSnip` | 最新点评节选（AI 参考；正文拉取时补全，初始 `''`） |

### 3.7 分类页签行（`parseTabGrid`，data.js:1498-1506）

字段同目录行的子集，`capacity/remaining = 0`（页签行无余量列，未知≠已满，
按需补拉会填），`partial: true`。

---

## 4. 志愿数据结构

### 4.1 volRow（`parseVolFromHtml` / `parseVolSportsFromHtml`，data.js:113-120/132-137）

`volMap` 的条目，键 = `code + '_' + normSeq(seq)`：

| 字段 | 说明 |
|---|---|
| `code` / `seq` | 课号 / 原始课序 |
| `department` | 开课系（错页校验用） |
| `capacity` / `applied` | 志愿口径容量 / 已报 |
| `volRequired` / `volElective` / `volOptional` | 必修/限选/任选志愿串（体育版只有 `volSports`） |

**墓碑行过滤**：容量与报名全 0 的行是已满课残影，解析时直接跳过（报名>0 的
0 容量行保留——超载是真信号）。

### 4.2 志愿串格式与解析产物

原始串：`'(1)2,4,5'` → `(N)` 前缀 = 优先志愿人数（第 0 档，任选走
`is_zyrxk=1` 通道），后跟从高到低各档人数。

`parseVolArr(s)`（probability.js:50）返回 `[一志愿, 二志愿, 三志愿]` 数组 +
`.priority` 附加属性：

- 3 个数 = 一/二/三志愿；1 个数 = 仅第三志愿开放（新生预选），缺的高档位补 0
  （右侧对齐）——绝不因数量不足判 null；
- 纯 `'(N)'` 无逗号串 → `[0,0,0] + priority=N`。

### 4.3 索引 map（`byCodeAll`，applyVolunteer 内）

`code → volRow[]`（同课号全部课序行），供"多段不盲配"核对：段对不上且该课
多段 → 宁缺毋滥返回 null；单段才允许回退取首行。

---

## 5. 课余量与队列数据

### 5.1 queueDataMap 条目（`fetchQueueData`，data.js:1187）

键 = `code + '_' + normSeq(seq)`：

| 字段 | 说明 |
|---|---|
| `code` / `seq` | 课号 / 原始课序 |
| `qCapacity` / `qRemaining` | 课余量口径容量 / 剩余 |
| `qQueue` | 排队人数（selectBksDlCount 批量接口回填，`dlrs` 字段） |

数据流：xkqkSearch 首页全量 grid 行 → 池内课程逐门 kylSearch POST 精确补充 →
排队人数批 100 查询（连败 3 批熔断并提示重登 WebVPN）。

### 5.2 已选页 zyMap（fetchSelectedCourses 内，data.js:1068-1075）

键 = `code + '_' + seq`（原始 seq）：

```js
{ zy: 志愿号, typeCode: '006'|'008'|'007', typeLabel: '必修'|… }
```

从页面脚本数组提取；体育课由"是"标记或空类型列推断。

### 5.3 levelMap 条目（`fetchLevelTable` / `fetchCategoryAttrs`，data.js:1364/1463）

键 = `code + '_' + normSeq(seq)`：

```js
{ typeCode, typeLabel, attr }   // 体育课 attr=''，typeLabel='体育'
```

`launch` 时 `levelMap = {...一级课表, ...分类页属性}`——**分类页（预选权威源）
优先**。

---

## 6. 暂存项与草稿对象

### 6.1 StageItem（`addToStage`，state.js:595-600）

```js
{
  code, seq, name, teacher,
  time, credits,
  flag,        // 提交类型 bx/xx/rx/ty（用户可选，受 allowedFlags 约束）
  zy,          // 志愿号 1-3
  baseFlag,    // 课程本身的类型（迁移/约束基准；旧数据缺失时迁移补 'rx'）
  note,        // 外校真实时间载体（课表预览 clockRangesOf 用）
}
```

### 6.2 Draft（`askReplaceDraft`，state.js:619）

```js
{ id: Date.now(), name, courses: StageItem[], createdAt: Date.now() }
```

上限 5 份；满时 prompt 输入编号替换。

### 6.3 导出 JSON 格式（`exportDraft`，state.js:675-681）

```json
{ "v": 1, "name": "草稿名",
  "courses": [ {code, seq, name, teacher, time, credits, flag, zy, baseFlag} ] }
```

### 6.4 ManualEvent（`showManualEventModal`，state.js:364）

```js
{
  id: Date.now(), name,
  code: 'manual-' + id, seq: '0',    // 伪课号供渲染管线统一处理
  day,        // 1-7
  begin, end, // 'HH:MM' 钟点（不限于大节）
  time: '',   // 恒空
  manual: true, credits: 0,
}
```

### 6.5 zyCache 条目（storage `zyCache`）

键 = `code_seq`：

```js
{ zy, typeCode, typeLabel,
  confirmed: bool,        // 用户在志愿确认弹窗亲手选过 = 永不再问
  confirmedAt?: number }
```

### 6.6 AI 课程 JSON（`aiCourseJson`，ai.js:11-28）

课程 → AI prompt 的统一投影：`name/code/seq/credits/teacher/department/time/attr/
typeLabel/remaining/capacity/available/selected/isCandidate/staged/zy/tongshiGroup/
courseFeature/grade/note/vol{capacity,applied,required,elective,optional,sports}/
reviewAvg/reviewCount/latestReview`（undefined 字段自动省略）。

---

## 7. 课表时间结构

### 7.1 时间槽位（`parseTimeSlots`，state.js:8）

返回 `[{day: '周一', slot: '1-2节'}]`。缓存于 `NX._slotsCache`（Map，时间串→槽位）。

### 7.2 钟点块（`clockRangesOf`，state.js:35）

外校课（北大/北外）时间解析产物：

```js
{ day,      // 1-7 数字
  begin,    // 开始分钟数
  end,      // 结束分钟数
  tag }     // '单周·1-16周' 等（单双周/周段标记，空=全周）
```

### 7.3 预览课块（renderPreviewTT 内 `mk()`，render.js:496-501）

```js
{ key,          // code_seq_tag 唯一键
  day, begin, end,          // 分钟
  label,        // '课名(教师)'
  color, probLabel, probBgColor,   // 着色与概率标签
  manual, id, code, seq, teacher, time,
  origin,       // 外校来源标签（NX.originOf）
  tag }         // 大节号 | 'clock' | 钟点解析 tag
```

### 7.4 分道信息（render.js:537-563）

`lanesOf: Map<块key, {lane, lanes}>`——簇内第几道 / 簇总道数，决定课块
`left/width` 百分比。

### 7.5 冲突报告（`detectConflicts`，state.js:99）

```js
{ day: '周三', slot: 时间或大节描述, a: 先出现课名, b: 后出现课名 }
```

`findPreviewConflicts` 返回 `{name, day, slot}[]`。

### 7.6 整体课表行（`parseWholeTimetable`，data.js:1685）

```js
{ code,        // 课号（个人整体课表内每门课一行）
  name,        // 课名（候选行去「候选：」前缀）
  teacher,     // 块内首属性（strHTML1 "；教师"）
  typeLabel,   // '必修'|'限选'|'任选'|''（体育块无类型属性）
  time }       // '天-大节(周次)' 逗号串，同 parseTimeSlots 格式
```

已选/候补行经 `overrideFromWholeTT`（data.js:1747）以此覆盖 time/teacher/name
（yxSearchTab 时间列脏数据正源修复，#46）；暂存且未选的课不在个人课表，
map 命不中，保持原数据链。

### 7.6 knote 条目（storage `knote`）

键 = `code_seq`：`{note, time}`——凡能解析就记（`knoteRemember` 400ms 防抖写盘），
预览 join 兜底用。

---

## 8. 评价层变量（reviews.js）

IIFE 内私有（未挂 NX 的），通过闭包共享：

| 变量 | 说明 |
|---|---|
| `S`（挂为 `NX.tbState`） | 评价层总状态对象，见下 |
| `S.ready` | 索引是否已建图 |
| `S.loadingPromise` | 单飞行加载 promise（防并发重复拉索引） |
| `S.entries` | 全部索引条目数组 `[{kcm, jsm, kkdw, sqid, tid, count, avg, nt}]` |
| `S.bySqid` | Map：`sqid → 条目` |
| `S.byNameT` | Map：`归一课名 + \\u0001 + 教师 token 键 → [条目]`（T1 精确匹配索引） |
| `S.byName` | Map：`归一课名 → [条目]`（T2 同名桶） |
| `S.detailCache` | Map：`sqid → {ts, data}` 正文内存缓存（10min） |
| `S.stats` | 匹配统计 `{t1, t2, t3, miss, total, matched}`（每次 tbAttach 重置） |
| `sgEl / sgItems / sgIdx / sgBlurTimer` | 联想词：下拉元素 / 候选课程数组 / 键盘选中索引 / 失焦延迟句柄 |

条目字段（`slimIndex` 精简后）：`kcm` 课名 / `jsm` 教师 / `kkdw` 院系 / `sqid`
课程 ID / `tid` 教师 ID（可空）/ `count` 点评数 / `avg` 均分（一位小数）/ `nt`
教师 token 数组。

归一化函数：`normName`（NFKC + 去全部空白 + 小写）、`normTeacherTokens`
（按 `,，、;；/\s` 拆 token）、`tKey`（token 排序后 `\u0002` 连接）。

---

## 9. 编码层变量（gbk.js）

| 变量 | 说明 |
|---|---|
| `GBK_CHARS` | GBK 字符表长字符串（OneTHU gbk-table.ts 逐字节移植） |
| `GBK_BYTES_B64` | 对应字节表的 base64 字符串 |
| `IDX` | 惰性 Map：`字符 → 表内索引`（`ensureTable` 首次调用时建） |
| `B1` / `B2` | 字节表解 base64 后对半拆分的高/低字节数组 |

`NX.gbkPercentEncode(s)`：纯 ASCII 直通；`%` → `%25`；CJK 查表输出
`%高字节%低字节`；表外字符回落 `encodeURIComponent`（UTF-8）。

---

## 10. 存储键清单（chrome.storage.local）

全部经 `NX.store.get/set`（自动加 `nextthuxk_` 前缀，配额失败 reject + 打日志）。

| 键 | 类型 | 写入处 | 说明 |
|---|---|---|---|
| `sem` | string | content.js launch | 当前学期 |
| `grade` | number | content.js launch | 年级（DATA_VER 升级时重置 0） |
| `staticData` | `{ver, plan, ts}` | content.js launch 收尾 | 仅存培养方案 + 版本；ver≠DATA_VER 整体清空（2.x 起不再缓存课程目录） |
| `stageCart` | StageItem[] | addToStage / removeFromStage 等 | 暂存区 |
| `drafts` | Draft[] | saveDraft / deleteDraft 等 | 草稿（≤5 份） |
| `zyCache` | `{code_seq: zyCacheEntry}` | resolveCourseZy | 志愿确认缓存（confirmed 永不再问） |
| `knote` | `{code_seq: {note, time}}` | knoteRemember（400ms 防抖） | 跨会话课表时间缓存（外校课时间记忆） |
| `manualEvents` | ManualEvent[] | showManualEventModal / removeManualEvent | 自定义时间占用 |
| `config` | `{api, model, token, pref}` | callAI 成功后 | AI 配置（含 token，仅本地） |
| `lastUpdateCheck` | number | checkUpdate | 上次检查时间戳（30 分钟节流；手动检查置 0） |
| `filtersOpen` | bool | content.js 筛选栏折叠 | 展开状态记忆 |
| `tbookIdx` | `{v, ts, courses}` | tbEnsureIndex | 社区评价精简索引（SWR） |
| `tbookIdxTs` | number | tbEnsureIndex | 索引时间戳 |

---

## 11. Shadow DOM 元素 ID 索引

`content.js` HTML 模板中的关键元素（`state.$(id)` 访问）：

| ID | 说明 |
|---|---|
| `nextthuxk-launch` | 右下角启动按钮 |
| `nextthuxk-toast` | 全局操作结果 toast |
| `nextthuxk-dashboard` | 全屏工作台容器 |
| `nextthuxk-inner` | Shadow 内容根 |
| `nextthuxk-search` / `nextthuxk-search-clear` | 搜索框 / 清空按钮 |
| `nextthuxk-suggest` | 联想词下拉（reviews.js 动态创建） |
| `nextthuxk-filters` / `nx-filter-toggle` / `nx-filter-body` | 筛选 chip 组 / 折叠按钮 / 折叠体 |
| `nx-filter-credits` / `-day` / `-period` / `-conflict` | 学分 / 周次 / 大节 / 冲突筛选下拉 |
| `nx-filter-reviews` / `nx-sort-by` | 社区评价筛选 / 排序下拉 |
| `nx-filter-tongshi` / `-feature` / `-grade-filter` / `-bksrem` / `-yjsrem` / `-xknote` | 通识组 / 特色 / 年级 / 本科余量 / 研院余量 / 文字说明筛选 |
| `nextthuxk-list` | 左栏课程列表容器（渐进渲染 + 分页条） |
| `nextthuxk-plan` / `nextthuxk-plan-detail` | 右栏培养方案进度卡 / 明细行 |
| `nextthuxk-preview-tt` / `nextthuxk-preview-info` / `nextthuxk-preview-reset` | 课表预览 / 标签 / 返回已选按钮 |
| `nextthuxk-add-manual` | 「＋ 添加占用」按钮 |
| `nextthuxk-queue-sec` / `nextthuxk-queue-list` / `nextthuxk-queue-count` | 候选队列区块 / 列表 / 计数 |
| `nextthuxk-stage-list` / `nextthuxk-stage-conflict` | 暂存区列表 / 冲突汇总 |
| `nextthuxk-draft-name` / `nextthuxk-save-draft` / `nextthuxk-save-selected` / `nextthuxk-preview-stage` / `nextthuxk-export` / `nextthuxk-import` | 草稿名输入 / 保存草稿 / 存当前选课 / 预览暂存 / 导出 / 导入按钮 |
| `nextthuxk-import-area` / `nextthuxk-import-data` / `nextthuxk-import-confirm` / `nextthuxk-import-cancel` | 导入区 / 文本域 / 确认 / 取消 |
| `nextthuxk-drafts` | 草稿卡列表容器 |
| `nextthuxk-api` / `nextthuxk-model` / `nextthuxk-token` / `nextthuxk-pref` | AI 配置四输入 |
| `nextthuxk-ai-search-prompt` / `nextthuxk-ai-search` / `nextthuxk-ai-search-st` / `nextthuxk-ai-search-results` | AI 搜索：描述框 / 按钮 / 状态 / 结果区 |
| `nextthuxk-ai` / `nextthuxk-ai-st` | AI 排课按钮 / 状态 |
| `nextthuxk-modal` / `-title` / `-body` / `-close` | 通用玻璃弹窗（简介 + 社区点评共用） |
| `nextthuxk-zy-modal` / `-body` / `-ok` / `-close` | 志愿确认弹窗 |
| `nextthuxk-drop-modal` / `nextthuxk-drop-title` / `-title2` / `-sub` / `-ok` / `-no` / `-no2` | 退选警告弹窗（玻璃红警示） |
| `nextthuxk-sem` / `nextthuxk-grade` / `nextthuxk-check-update` / `nextthuxk-exit` | 顶栏：学期 / 年级 / 检查更新 / 返回原系统 |
| `nextthuxk-cache-info` | 顶栏缓存/同步信息条 |
| `nextthuxk-phase-tag` | 「课余量模式」阶段标签 |
| `nextthuxk-update-banner` / `nextthuxk-danger-banner` | 更新 / 危险横幅（update.js 动态注入） |
| `nx-build-tag` | 构建标记（rt-park1） |

---

## 12. 模块级杂项变量

| 变量 | 位置 | 说明 |
|---|---|---|
| `_knoteSaveT` | state.js:254 | knote 写盘防抖句柄 |
| `NX._slotsCache` | state.js:11 | 时间串 → 槽位解析缓存 Map |
| `NX._lowerCache` | state.js:343 | 小写化缓存 Map（`NX.lc` 用；全校课名种类有限无泄漏风险） |
| `NX._selCache` / `NX._selCacheV` | state.js:304 | 已选预览行缓存 / 版本键（`selVersion\|候选数\|poolVersion`） |
| `NX._pvRef` / `NX._pvLen` / `NX._pvIdx` | state.js:313 | 槽位索引三重失效：源数组引用 / 长度签名 / 索引本体（`invalidatePreview` 清） |
| `NX._ssTimer` | render.js:1609 | 服务端搜索防抖句柄（500ms，immediate=0ms） |
| `NX._sgBlurT` | content.js:621 | 联想词失焦延迟句柄 |
| `NX._undetBfT` | render.js:531 | 时间未定回填防抖句柄 |
| `NX._probBfTimer` | render.js:776 | 概率回填防抖句柄（700ms） |
| `HTML` / `cssText` | content.js:34/43 | 工作台 HTML 模板字符串 / 拉取的样式文本 |
| `WEBVPN_PREFIX_RE` | content.js:29 | `/^\/(https?)\/([0-9a-f]{32,})(?=\/|$)/i`，WebVPN 路径前缀正则 |
| `_browser` / `statusEl` / `launchBtn` | popup.js | 弹出页：浏览器 API / 状态行 / 启动按钮 |

---

*文档生成于 v2.1.1（构建 rt-park1）。与 ARCHITECTURE.md 配套阅读。*
