需要在HTML代码中加入SHA256密钥和API密钥.
姓名等信息请自行加入.

# 27.4 重实现规格说明

> 分析对象：`点名网页本地版 27.3.2 AI Openrouter SHA-256 哈希+双重验证加密+本地储存.html`（共 4917 行，`<script>` 内容自约行 1200 起）。
> 本文为只读分析结果，供全新重写 27.4 版并保持**行为一致**使用。所有行号均指向源文件，引用以「行 N」标注。

---

## 0. 全局结构与重要认知（先读）

脚本整体包在一个大型 IIFE 中（结尾 `})();` 于行 4902）。页面含大量**反篡改/自毁机制**（`_metaGuardFlag`、`_securityCheckPassed`、`sessionStorage.__meta_flag`、`selfDestruct()`、devtools 检测、`localStorage/sessionStorage.clear()` 等，行 2314~2542、2650~2706、3022~3034、2485~2537）。若要"保留相同行为"，这些守护逻辑也需照搬，否则页面会自我清空/关窗。本文不逐条展开守护代码，但提醒重实现时不可省略 `init` 前的元防护初始化。

页面 UI 是"玻璃拟态 dashboard"，主要区块由 `#main-dashboard` 与 `#ai-drawer`(AI聊天抽屉) 通过 `.menu-tab` 切换（`switchTopPanel` 行 4635）。核心点名控件在 dashboard 顶部玻璃面板（行 ~1456 起的 HTML）。

### 两套"模式"必须分清（用户术语与本代码的映射）

| 概念 | 变量 | 取值 | 设置入口 |
|---|---|---|---|
| 大模式(顶部) | `currentMode` | `'pick'`(点名) / `'group'`(分组) | `.mode-option[data-mode]`，`setMode()` 行3622 |
| 抽取权重模式 | `currentRollMode` | `'normal'` / `'ranking'` / `'ai'` | `normal/ranking/ai-roll-mode-btn`，`setRollMode`→`switchToRollMode` 行2601 |
| 页面UI class | `currentUIMode` | `normal/ranking/ai` | `applyUIMode()` 行2306（body class `mode-normal/mode-ranking/mode-ai`） |
| 排名开关 | `isRankingMode` | bool | `setProbabilityMode()` 行3624 |

> 代码中**不存在**字符串 "random" / "pickone" 作为模式值。用户所述 "normal/random/pickone/AI" 应拆为：`currentMode` 的 pick/group，加 `currentRollMode` 的 normal/ranking/ai，组合出行为。抽取本身有"点单次、点多人、随机抽、AI智能点名、语音点名、随机分组/抽一组"等不同入口（见第 3 节）。

---

## 1. 数据模型与默认名单

### 1.1 CONFIG 常量 —— 行 2158-2173
```js
const CONFIG = {
    STORAGE_KEY: 'class-picker-v2-group',   // 主存储键
    MAX_WEIGHT: 10,                          // 权重上限(但代码多处用 10.0 截断)
    MIN_WEIGHT: 0.1,                         // 权重下限
    DEFAULT_MAX_STUDENTS: 500,               // 最大人数默认
    MIN_MAX_STUDENTS: 1,                     // 高级设置可设下限
    MAX_MAX_STUDENTS: 20000,                 // 高级设置可设上限
    MAX_SELECTIONS: Number.MAX_SAFE_INTEGER, // 被点次数/总抽取的“理论”上限(实际无上限)
    MAX_HISTORY: 10000,                      // history 最大条数(超出截尾)
    BIG_NUMBER_THRESHOLD: 1000000,
    BIRTHDAY_FEATURE_KEY: 'birthday_feature_enabled', // 生日提醒开关存储键
    AI_MATCH_THRESHOLD: 0.62,                // 语音姓名匹配最低相似度
    AI_PROB_INCREMENT: 0.1,                  // AI 点名每次 +0.1 权重
    ANIMATION_FRAMES: 24,                    // start 滚动抽帧动画帧数
    CONFETTI_COUNT: 20                       // 庆祝彩屑数量
};
```
- 其他顶层常量：
  - `globalMaxStudents = CONFIG.DEFAULT_MAX_STUDENTS;`（行 2175）——**仅内存，不落盘**，见 2.4。
  - `IDCARD_PASSWORD_HASH`（行2176）、`EXPECTED_HASH_1`(行2177)、`EXPECTED_HASH_2`(行2178)：均为 SHA-256 hex。双重验证密码一、二层。
  - `RANKING_WEIGHTS`（行2180-2191）：按姓名→权重表，权重取值如 `10, 9.5, 9.0, 8.5, 8.0, 7.5, 7.0, 6.5, 5.0, 4.0, 3.5, 3.0, 2.5, 2.0, 1.0`。49 名学生几乎全覆盖。

### 1.2 appData（内存主数据）—— 声明行 2287，结构与 DataManager 出入一致
`let appData = DataManager.load(), ...`（行2287）。实际形状（load 的行 2263-2275）：
```json
{
  "students": [ 学生对象... ],
  "totalSelections": 0,      // 总抽取次数
  "history": [ {time,names[],source?} ]
}
```
> 其它全局变量（行 2286-2304）：`_metaGuardFlag`, `isAnimating`, `isDevOpen`, `currentMode='pick'`, `failedAttempts`, `MAX_FAILS=15`, `passwordStep`, `currentPasswordHash`, `lastGroups=[]`, `idcardUnlocked`, `isRankingMode=false`, `birthdayEnabled=true`, `birthdayFireworkInterval`, `birthdayAnimationInterval`, `recognition`, `recognitionActive`, `recognitionSupported`, `currentRollMode='normal'`, `currentUIMode='normal'`。计时器类：`timerMode`, `timerRunning`, `timerStartTime`, `timerPausedTime`, `timerTargetSeconds`, `timerRemainingSeconds`, `beepPlayedForFiveSec`, `audioCtx`, `localTimeInterval`, `animationFrameId`（行2707-2716）。

### 1.3 学生对象字段
`defaultStudents()`（行2209-2259）每项含 6 字段：
```js
{ name:'陈钰泽', gender:'男', idCard:'330127201202250036', weight:10, selectedCount:0, id:'s1' }
```
- `name` 汉字姓名（可含数字/字母，见第5节校验）。
- `gender` `'男'/'女'`，新建默认 `'未知'`。
- `idCard` 18 位身份证（部分为假号如 `000000201203130000`；birthday 功能从中截取月日）。
- `weight` 权重（默认数组内为 10，但 `余泽楷` 为 9.9，行2229——**init 时会被覆盖成 10**）。
- `selectedCount` 被抽中累计次数。
- `id` 稳定标识；默认名单为 `s1..s49`，新增用 `genId()`（行3053）：`'s'+Date.now()+'_'+Math.random().toString(36).substr(2,9)`。

默认名单共 **49 人**：男 24（s1-s24），女 24（s25-s48），徐千城俊(s49)男。姓名清单与 RANKING_WEIGHTS 键一致。默认全部 weight=10、selectedCount=0。

### 1.4 权重的单位/含义（重点，含“与10%的关系”）
代码对 weight 有两套并存尺度，重实现需原样保留：

1. **内部原始权重** `student.weight`：多为 0.1~10 的浮点（RANKING 用 1.0~10；AI 模式默认 1.0；正常/默认模式 10）。
2. **UI “百分比%”** = `weight * 10`。即：
   - weight=10 ⇔ 显示 100%（满）
   - weight=1.0 ⇔ 显示 10%（resetAllW/equalizeW/AI 默认口径）
   - weight=0.1 ⇔ 显示 1%
   换算函数 `updateWeightDisplay(index, percent)` 行3210-3211：`const weightVal = percent / 10; appData.students[index].weight = weightVal;`，即把 UI 百分数转回 weight。`resetAllW`(行3693) confirm“重置所有权重为10%?” → 对所有学生 `updateWeightDisplay(i,10)` → weight=1.0。`equalizeW`(行3694)：`avgPercent = Math.round(avgWeight*10)`。
3. **真正的“被抽概率”** = `weight / Σweight × 100%`，见 `calcProb()`(行3207) 与抽取算法（抽时对每个 weight 用 `Math.max(0.01, weight)` 兜底，行2670/2674/3331/3334）。因此 49 人且权重全 1.0 时，每格实际概率≈2%，并非 10%——UI 上“%”多指 weight×10 的“权重刻度”，非真实概率。消息文案（如“每人默认最低概率10%”“+1%”）是粗略说法。

> 反差提醒：`defaultStudents` 与 `setProbabilityMode(false)` 写 **weight=10(=100%刻度)**；但 `resetAllW` 写 **weight=1.0(=10%刻度)**；`switchToRollMode('ai')` 写 **weight=1.0**。三者并存且互不一致，请按原样复现（含 UI 满条/10% 的外观差异）。

### 1.5 生日相关（涉及学生数据使用，简要）
`extractBirthdayFromIdCard`(行3056) 从18位取 `[6,12)`、15位取 `19+[6,12)`。`getTodayBirthdayStudents`(行3073) 过滤今日生日者，`showBirthdayBanner`(行3164) 显示横幅+`startBirthdayFireworks`(行3137) 烟花。开关存于 `CONFIG.BIRTHDAY_FEATURE_KEY`(`birthday_feature_enabled`)，init 时读取(行4788)。

---

## 2. StorageManager 与 DataManager

### 2.1 StorageManager —— 行 2193-2207
存储检测与读写适配层：
```js
const StorageManager = {
    storageType: 'none', memory: {},
    check() {  // 依次探测 localStorage → sessionStorage → memory
        try{ localStorage.setItem('test','1'); localStorage.removeItem('test'); this.storageType='localStorage'; return true;}catch(e){}
        try{ sessionStorage.setItem('test','1'); sessionStorage.removeItem('test'); this.storageType='sessionStorage'; return true;}catch(e){}
        this.storageType='memory'; return false;
    },
    get() { return this.storageType==='localStorage'?localStorage
                : this.storageType==='sessionStorage'?sessionStorage : this.memory; },
    getItem(k){ const s=this.get(); return s[k]||null; },   // 读原样字符串
    setItem(k,v){ try{ const s=this.get(); s[k]=v; return true;}catch(e){return false;} }, // 写字符串
    removeItem(k){ const s=this.get(); if(this.storageType==='memory') delete s[k]; else s.removeItem(k); },
    info(){ /* 返回 {text,icon}：本地永久存储💾 / 临时会话存储⏳ / 内存存储(刷新丢失)⚠️ */ }
};
```
- 探测优先级 `localStorage > sessionStorage > memory`；`check()` 在 init 行4680 最先调用。
- `getItem` 当值为空/不存在返回 `null`；**存的是原始字符串，不做自动反序列化**（JSON.parse 在调用方做）。
- init 行4784-4786：`inf=StorageManager.info()` 写入 `#storage-warning-text`；非 localStorage 时显示 `#storage-warning`。

### 2.2 DataManager —— 行 2262-2285
```js
const DataManager = {
    load(){
        try{
            const d=StorageManager.getItem(CONFIG.STORAGE_KEY);   // 键 'class-picker-v2-group'
            if(d){
                let p=JSON.parse(d);
                if(p.students) p.students.forEach((s,i)=>s.id=s.id||'s'+Date.now()+'_'+i); // 缺id补齐
                p.totalSelections = p.totalSelections || 0;
                if (p.totalSelections > CONFIG.MAX_SELECTIONS) p.totalSelections = CONFIG.MAX_SELECTIONS;
                if (p.history && p.history.length > CONFIG.MAX_HISTORY) p.history = p.history.slice(-CONFIG.MAX_HISTORY);
                return p;
            }
        }catch(e){}
        return {students:defaultStudents(), totalSelections:0, history:[]}; // 无数据→默认名单
    },
    save(d){
        try{
            if (d.history && d.history.length > CONFIG.MAX_HISTORY) d.history = d.history.slice(-CONFIG.MAX_HISTORY);
            return StorageManager.setItem(CONFIG.STORAGE_KEY, JSON.stringify(d)); // JSON 序列化后存字符串
        }catch(e){ return false; }
    },
    isDev(){ return StorageManager.getItem('dev_auth_off')==='true'; },   // 高级面板未锁
    setDev(v){ v?StorageManager.setItem('dev_auth_off','true'):StorageManager.removeItem('dev_auth_off'); }
};
```
- **主数据键名**：`CONFIG.STORAGE_KEY = 'class-picker-v2-group'`，值为 `JSON.stringify(appData)` 整串。
- load 时若解析/读取失败或无值 → 返回默认名单（totalSelections=0, history=[]）；缺 id 补 `'s'+Date.now()+'_'+i`。
- `MAX_HISTORY=10000` 上限在 load 与 save 双端都截尾。
- 其它存储键：`dev_auth_off`（dev未锁定标记）、`debug_panel_enabled`、`birthday_feature_enabled`、`whatsnew_version_27.3.2_minimax_m2.7_v3`、`your-API-key-here`（OpenRouter key 存储键，常量名 `OPENROUTER_KEY_STORAGE` 行2920）、`openrouter_model_selection`（行2921）。

### 2.3 dev 权限（两层密码）
- 仅当 `DataManager.isDev()` 为真才直接 `openDev()`；否则走 `openPwd()`→`verifyPwd()`(行3663)：第一层 SHA-256 等于 `EXPECTED_HASH_1` 进第二层，等于 `EXPECTED_HASH_2` 则 `DataManager.setDev(true)` 并开面板。连续失败 `MAX_FAILS=15` 触发 `selfDestruct`。
- 身份证查看单独密码：`verifyIdcardPassword`(行3664)，哈希 `IDCARD_PASSWORD_HASH`，成功后 `idcardUnlocked=true` 显示 idcard 表格。

### 2.4 globalMaxStudents 从哪读
`globalMaxStudents` **不持久化**。来源只有行2175 `CONFIG.DEFAULT_MAX_STUDENTS`(500)；init 行4787 仅把它写进 `#max-storage-limit` 输入框并绑定 `updateMaxStorageLimit`(行3692)。修改只改内存变量，刷新即回 500。凡"超过 X 人"校验（saveNames 行3619、confirmImp 行3697）都用此值；校验范围 `MIN 1 ~ MAX 20000`，不能小于当前人数。

---

## 3. 点名核心逻辑与所有抽取模式

### 3.1 抽取工具函数
- `weightedSelect(count)` 行3566（无放回加权抽取，直接累加 `s.weight`，无 0.01 兜底）：
  ```
  avail=[...students]; 每轮 total=Σweight; r=Math.random()*total; 顺减，r<=0 取中并从 avail 移除
  ```
- `aiWeightedSelect(count)` 行3326（同逻辑但 `Math.max(0.01, Number(weight||0))` 兜底），供 runAiRoll。
- `rollOneStudentByCurrentMode()` 行2661-2698（语音"开始点名"用）：normal→`randomIndex` 均匀随机；ai/ranking→加权轮盘（`Math.max(0.01,…)`）；命中后 `selectedCount=safeAdd(+1)`、`totalSelections=safeAdd(+1)`、push history(`source:'voice_cmd'`)、save、`showResult([selected])`、`updateStats/renderList/renderDev`、`confetti()`、`highlightStudentInList`。空名单 `showWarn('名单为空')`。
- `safeAdd(a,b)` 行3036：和超 `CONFIG.MAX_SELECTIONS` 则截断（实际 = Number.MAX_SAFE_INTEGER，等于无上限）。
- `highlightStudentInList(studentName)` 行2543-2580：找到 `.student-chip` 内 `.student-name` 文本匹配，`scrollIntoView(center)`，加 `highlight-student` + 金色边框/光晕 3 秒后还原并回滚 `window.scrollY`。
- `confetti()` 行3569：`CONFIG.CONFETTI_COUNT`(20) 片，间隔 20ms，色板 `['#7bb3e0','#6ba5d9','#9ac4ed','#b0d4ff']`，2.5s 后移除。

### 3.2 大模式切换 setMode(mode) —— 行3622（与 UI 显隐）
- `currentMode=mode`；高亮 `.mode-option.active`。
- `mode==='group'`：显示 `#group-controls`(flex)，隐藏 `#start-btn` 与 `#student-count-container`。
- 否则：隐藏 group-controls，显示 start-btn(inline-flex) 与 student-count-container。
- HTML 大模式只有 `data-mode="pick"/"group"`（行1460-1462）；init 行4892 `setMode('pick')`。

### 3.3 抽取权重模式 switchToRollMode / setRollMode —— 行2601-2649、3375
`setRollMode(mode)`(行3375) 即 `switchToRollMode(mode)`+devLog。`switchToRollMode` 动作：
- `mode==='ai'`：**把所有学生 weight 置 1.0**（行2604-2606），状态文案“AI智能点名模式”，`applyUIMode('ai')`；消息“每人默认最低概率10%，每被语音点名一次概率+1%”。
- `mode==='ranking'`：`setProbabilityMode(true)`，`applyUIMode('ranking')`。
- 否则(normal)：`setProbabilityMode(false)`，`applyUIMode('normal')`。
- 按钮激活态：normal→`.btn-primary`，ranking→`.btn-ranking`，ai→`.btn-primary`，其余一律 `.btn-soft`。
- 结尾 `DataManager.save(appData); if(isDevOpen)renderDev(); renderList(); updateStats();`（行2645-2648）。
- init 行4828 设 `currentRollMode='normal'` 并 `applyUIMode('normal')`；行4830-4832 绑三按钮。

### 3.4 主“开始点名” start() —— 行3582-3612（pick 模式，含动画）
1. `if(isAnimating)return;`
2. `if(currentMode==='pick')`：
   - **若 currentRollMode==='ai' → 直接 `runAiRoll()` 并 return**（不走下面动画）。
   - `cnt=parseInt(els.studentCount.value)`（下拉 1~10，HTML 行1500-1510）。
   - 名单空→`showWarn('名单为空')`；`cnt>len`→`showWarn('最多X人')`。
   - `isAnimating=true`，start 按钮禁用，文字“⏳ 抽取中...”。
   - `setInterval(...,40)` 滚动预览帧，每 40ms 一次、共 `ANIMATION_FRAMES`(24) 帧：每帧从学生池随机(无放回)取 cnt 人 `showResult(tmp)` 做视觉滚动。
   - 到 24 帧：`clearInterval`，`final=weightedSelect(cnt)`（真实加权落定，无放回）；遍历 final 每人 `selectedCount=safeAdd(+1)`、`totalSelections=safeAdd(+1)`；`history.push({time:iso,names:final.map(name)})`（无 source）；`DataManager.save`；`showResult(final)`；`updateStats/renderList/(renderDev)/confetti`；逐人 `highlightStudentInList`；恢复按钮“🎯 开始点名”；`devLog('点名: ...(模式: 成绩排名概率/正常点名)')`——**此处 devLog 依据 isRankingMode 判定**。
3. 否则 `showWarn('请在分组模式使用分组按钮')`。

> 关键差异：主按钮落定用 `weightedSelect`（weighted、无兜底），与 ai 模式改走 `runAiRoll`（`aiWeightedSelect`，有兜底）不同。

### 3.5 AI 智能点名 runAiRoll() —— 行3345-3373
- `if(isAnimating)return;` 名单空→warn；start 按钮文字“🤖 AI 分析中...”。
- `count=parseInt(els.studentCount.value)||1`；`selected=aiWeightedSelect(count)`（加权随机，**不真正调用 OpenRouter 选人**）。
- 每人 `selectedCount+1`、`totalSelections+1`；`history.push({...,source:'ai'})`；save；`showResult(selected)`；`updateStats/renderList/renderDev`；`confetti`；恢复按钮“开始点名”；逐人 highlight；devLog“AI点名: 名单”。
- 说明：所谓"AI 智能点名"本质是**自适应加权随机**：切 AI 模式权重全置 1.0，之后凡语音点到某人则 `increaseAIWeightForStudent` 使该人权重 +0.1（上限 10.0），从而概率向被点过的学生倾斜。

### 3.6 setProbabilityMode(useRankingMode) —— 行3624-3658（权重分配核心）
```
isRankingMode = useRankingMode;
LOWEST_STUDENTS = ['杨子霂','骆清昊'];
ranking 模式：每学生 weight = RANKING_WEIGHTS[name] ?? 0.1；随后 LOWEST 两名强制 0.1；
normal 模式：LOWEST 两名 → 0.1，其余 → 10；
状态条文案 + .ranking-active class；devLog/showWarn；
DataManager.save(appData); renderDev; renderList; updateStats;   // 每次切换/调用都会落盘
```
> 注意：normal 分支 weight=**10**（满刻度），ranking 用表内 1.0~10，LOWEST 恒 0.1。rank 外的人(表外)给 0.1。

### 3.7 语音点名 rollOne（见第7节）与展示函数
- `showResult(selected)` 行3567：隐藏 `#result-placeholder`、显示 `#result-content`、隐藏 `#group-result`；为每人渲染 `.result-card`：姓名首字圆形头像（60px，字号2rem）+ `<h3>`姓名 + “被抽中 X 次”(formatBigNumber) + 右侧 `#序号`。
- `showGroups(groups,isPickOne)` 行3568：非 pickone 时写 `lastGroups`；渲染玻璃组卡片“第 N 组 (M人)”，成员为胶囊名。用于随机分组/抽取一组。
- `formatBigNumber` 行3035：`>=1e6→'1.0M'，>=1e3→'1.0K'，否则原样`。

### 3.8 分组/抽组（currentMode=group）
- `randomGroup()` 行3614：校验 `#group-size`（`validateGroupSize` 行3570：正整数且 1≤≤学生数）；Fisher-Yates 打乱；按 size 切片 `groups`；`showGroups`；`confetti`。
- `randomPickGroup()` 行3615：无 `lastGroups`→warn“请先进行随机分组”；随机取一组 `showGroups([…],true)`。
- group 模式按钮只在 group 模式可见（HTML 行1472-1473 的 `#random-group-btn/#random-pick-group-btn`）。

### 3.9 Agent 工具触发的点名（AI 聊天/agent 模式）
`executeTool` 行3958-4107 与 `AGENT_TOOLS` 行2723-2919：
- `execute_action.start_roll_call`/`random_pick` → `els.startBtn.click()`（即触发 `start()`）。
- `reset_roll_call` → 点 `#reset-stats`；`reset_all` → 点重置 + `StorageManager.clear()` + `location.reload()`。
- `get_page_info` → `getStudentList()`(行3945，只回 name/gender/selectedCount/weight) + `getRollCallStats()`(行3948，回 total_students/total_selections/times_selected/most_selected/history_count)。
- **已知缺陷**：`open_student_edit` 与 `execute_action.edit_students`、`clear_selected` 调用未定义的 `openEditModal()`/`clearSelected()`/`isDevLocked`（全文件无定义），agent 下会抛 ReferenceError（被 sendChatMessage 捕获显示"请求失败…is not defined"）。真正名单编辑走 UI `openEdit/saveNames`（见第5节）。
- agent 模式开关 `#agent-mode-switch`（init 行4878），仅聊天附加工具执行，不影响点名核心。

---

## 4. 名单渲染与统计

### 4.1 renderList() —— 行3183-3191（主页面学生列表）
```html
<div class="student-chip flex justify-between">
  <div><span class="student-name" style="font-weight:700">姓名</span><br>
       <small>抽中 {formatBigNumber(selectedCount)}</small></div>
  <div style="font-size:2rem">{formatBigNumber(selectedCount)}</div>
</div>
```
- 写入 `#student-list`(`els.studentList`)。**这里没有权重滑块/概率条**（那在 dev 面板 renderDev）。首字母头像、卡片式信息是 `showResult` 的事。

### 4.2 updateStats() —— 行3193-3205
- `#total-students`=学生数；`#selected-count`=`formatBigNumber(totalSelections)`；`#selected-students`=`filter(selectedCount>0).length`；`#max-selected-count`/`#max-selected-name`=被点次数最多者（线性扫最大值；最高 0 时也取首人）。

### 4.3 renderDev() —— 行3235-3297（仅当 isDevOpen，dev 面板内）
两个子面板：
1. `#probability-overview`(`els.probOverview`) 每学生一行（行3239-3248）：
   - 名字、可拖拽 `.prob-bar-container`（高12px，橙色 `.prob-bar-fill`，宽=`weight*10%`，圆角，cursor:ew-resize，右端 `.drag-handle`）、右侧 `#prob-text-i`（**真实概率** `s.prob%`，来自 `calcProb`）。
   - 拖拽/点按：`onMouseMove` 按容器宽度算 `percent=clamp(round(x/width*100),1,100)` → `updateWeightDisplay(idx,percent)`；mousedown/move/up 绑定 document。
2. `#dev-student-weights`(`els.devWeights`) 每学生（行3281-3288）：
   - 名字 +(selectedCount) + `.weight-percent-value`(percent=weight*10 显示 `%`)、`<input type=range min=1 max=100 step=1>`(`.dev-slider`)（input→`updateWeightDisplay`）、快捷按钮 `[5,10,20,50,100]%`（调 `window.qsWeight(i,v)`，行3299）。
- 辅助：`calcProb()` 行3207 返回 `{...s, prob: t>0?(weight/t*100).toFixed(1):0}`；`getProbClass(p)` 行3208：`≥20 'prob-high'，≥10 'prob-med'，else 'prob-low'`。

### 4.4 updateWeightDisplay(index, percent) —— 行3210-3233（写权重的统一通道）
```
weightVal = percent/10; 学生.weight=weightVal; DataManager.save(appData);   // 每次改权重即保存
#prob-item 内 .prob-bar-fill 宽=percent%；.prob-percent 文字=percent%
#weight-item 内 .weight-percent-value=percent%；slider.value=percent
再算 total=Σweight；newProb=weight/total*100 .toFixed(1)；写 #prob-text-i = newProb%
devLog('调整 name → percent%')
```
### 4.5 renderIdCards() —— 行3301-3324（dev 面板 idcards 分页，需 `idcardUnlocked`）
- `#id-card-tbody` 行：姓名 / gender(默认'未知') / 身份证(空则灰字“无”) / 编辑按钮 `✏️`。
- 编辑：`prompt('请输入新的身份证号', 现值)`，非 null → 写 idCard、`DataManager.save`、重绘、`showBirthdayBanner`、devLog。
- init 行4793/4805：birthday 开关/关闭横幅。

---

## 5. 名单编辑

### 5.1 打开/关闭 openEdit / closeEdit —— 行3616-3617
- `openEdit()`：`#edit-modal` 去掉 `.hidden`；`#names-input.value = students.map(name).join('\n')`（一列一个）；`window.scrollTo(top,0,smooth)`。
- `closeEdit()`：加 `.hidden`。触发：`#edit-names`(行4840)、`#close-edit-modal`、`#cancel-edit`、点模态背景（行4857）。`window.closeEdit=closeEdit` 行3618 暴露全局。

### 5.2 saveNames() —— 行3619（保存核心，含校验+合并）
处理流程（原文）：
1. `names = namesInput.value.split('\n').map(trim).filter(Boolean)`。
2. **人数上限**：`names.length > globalMaxStudents` → warn“编辑失败：学生人数不能超过X人（可在高级设置中调整）”并 return。
3. 逐名校验：
   - `isNameLengthValid(name)`(行3038)：`name.length<=6`，否则“超过6个字符限制”。
   - `isValidName(name)`(行3037)：`/^[\u4e00-\u9fa5a-zA-Z0-9]+$/`，否则“包含非法字符，只允许英文字母、中文汉字和数字”。
4. 空名单 → “至少一个姓名”。
5. **新老合并**：对每个新名在旧 `appData.students` 找同名 `old`；构造
   ```js
   { id: old?old.id:genId(),
     name: n,
     gender: old?old.gender:'未知',
     idCard: old?old.idCard:'',
     weight: old?old.weight:1,        // 旧学生保留原 weight；新人默认 1（非10！）
     selectedCount: old?old.selectedCount:0 }
   ```
6. `appData.students = newStu`；**随后 `setProbabilityMode(isRankingMode)`**（会按当前模式重写所有人的权重并 save）；`DataManager.save(appData)`；`renderList/(renderDev)/updateStats`；`closeEdit`；devLog“名单更新: N人”；showWarn“✅ 已保存N人”；`showBirthdayBanner`。
> 重要：新建学生默认 weight=1（=10% 刻度），但保存紧接着的 `setProbabilityMode(false)`(正常模式) 会把所有人都写成 10（LOWEST 除外），因此实际落盘权重多半被覆盖为 10/0.1，取决于是不是 ranking。

### 5.3 校验函数定义（行3036-3054）
- `isValidName`：`/^[\u4e00-\u9fa5a-zA-Z0-9]+$/`（中英数字，无符号/空格/下划线）。
- `isNameLengthValid`：`len<=6`。
- `genId`、`escapeHtml`、`formatBigNumber`、`safeAdd`、`sha256`(WebCrypto)、`showWarn`(3s 自动隐藏)。
- `showWarn` 行3040：写 `#warning-text`，显示 `#warning-message`，3000ms 后加回 `.hidden`。

### 5.4 导入/导出（旁路名单变更）
- `exportD` 行3695：下载 `点名数据_日期.json`，`JSON.stringify(appData,null,2)`。
- `importD` 行3696 弹窗；`confirmImp` 行3697：parse；须 `d.students` 是数组；`length<=globalMaxStudents`；逐学生补齐缺省并校验：缺 `id`→`genId()`；`weight=s.weight||1`；`selectedCount||0`；`idCard||''`；`gender||'未知'`；`selectedCount>MAX→MAX`；`isValidName/isNameLengthValid` 不过抛 `throw`。然后 `appData=d`、`setProbabilityMode(isRankingMode)`、save、render、showBirthdayBanner；catch→showWarn“❌ 导入失败: …”。

---

## 6. 权重操作与 ranking

### 6.1 updateWeightDisplay —— 见 4.4（唯一写单权重入口，含 drag/slider/快捷按钮）。

### 6.2 resetAllW() —— 行3693
`confirm('重置所有权重为10%?')` → 逐人 `updateWeightDisplay(i,10)`（weight→1.0）。showWarn“权重重置为10%”。

### 6.3 equalizeW() —— 行3694
`avgWeight=Σweight/n`；`avgPercent=Math.round(avgWeight*10)`；逐人 `updateWeightDisplay(i,avgPercent)`（均摊）。

### 6.4 setProbabilityMode —— 见 3.6（ranking vs 非 ranking 是它的两分支）。rankingMode/非 ranking 差异汇总：
| | ranking(true) | normal(false) |
|---|---|---|
| LOWEST 两人(杨子霂/骆清昊) | 0.1 | 0.1 |
| 其余命中 RANKING_WEIGHTS | 表中值(1.0~10) | 10 |
| 其余(表外) | 0.1 | 10 |
| 状态条 | 当前:成绩排名概率模式 + .ranking-active | 当前:正常点名模式 |
| 触发 | normal/ai 按钮之外的 ranking 按钮→switchToRollMode('ranking') | init、normal 按钮、save/import/reset 后 |

### 6.5 AI 权重自适应 increaseAIWeightForStudent(studentName) —— 行2582-2599
- 找 name 学生；仅当 `currentRollMode==='ai'` 才生效：
  - `newWeight = weight + CONFIG.AI_PROB_INCREMENT(0.1)`；`>10.0` 截到 `10.0`；`weight=parseFloat(newWeight.toFixed(2))`；`DataManager.save`。
  - 计算 `actualProb=(weight/Σweight*100).toFixed(1)`；devLog“AI智能点名模式：X 概率提升1%（当前权重…，概率…%）”；showWarn“📈 X 点名概率 +1%”；renderDev/renderList。
- 只在 `selectStudentByName`(行3438-3440) 的语音点名中调用（AI 模式下每次点到即提权）。
- `highlightStudentInList` 见 3.1。

### 6.6 dev 面板锁定流
`openDev`(行3669)/`closeDev`(行3680)/`lockDev`(行3690)/`toggleDev`(行3691)。`#max-storage-limit` 只读内存（2.4）。权重重置/均衡/导出/导入按钮绑 dev 面板（init 行4850-4853）。`renderDev` 仅在 `isDevOpen` 时画；dev 面板打开须过两层密码（见 2.3）或 dev_auth_off。

---

## 7. 语音点名（名字匹配与命令）

### 7.1 语音入口与生命周期
- `#mic-btn`(els.micBtn) → `startVoiceRecognition()` 行3553：
  1. `!recognitionSupported` →“当前浏览器不支持语音识别功能”。
  2. 若 `recognitionActive`：`recognition.stop()`、复位按钮文字“🎤 语音识别点名”、“🔴 已停止”。
  3. `requestMicrophonePermission()` 行3446：`navigator.mediaDevices.getUserMedia({audio:true})` 后即 `stop()` 各轨；失败 showWarn“请允许麦克风权限后重试”。
  4. 无 recognition 则 `initSpeechRecognition()`；`recognition.start()`；异常则重建再试。
- `initSpeechRecognition()` 行3459：`SpeechRecognition||webkitSpeechRecognition`；不支持则“当前浏览器不支持”；实例配置 `lang='zh-CN', continuous=true, interimResults=true, maxAlternatives=3`。回调：
  - `onstart`：按钮“🎤 🟢 监听中...”，showWarn“持续监听中...”。
  - `onerror`：`no-speech` 静默忽略；`not-allowed`/`audio-capture` 专门文案；其余“识别失败:err”。失败后 `recognitionActive=false`、按钮复位。
  - `onend`：若按钮仍含“🟢”，200ms 后自动 `recognition.start()` 重启持续监听；否则复位。
  - `onresult`：取 `isFinal` 优先的整段 transcript（无 final 取首个 interim），空则忽略。
- init 行4829 于加载即 `initSpeechRecognition()`（探测支持性）。

### 7.2 文本处理/匹配函数
- `normalizeTranscript(text)` 行3380：`toLowerCase()` 去除非中英数字（`/[^\u4e00-\u9fa5a-z0-9]+/g`→''）。
- `similarityScore(a,b)` 行3384-3395：都对归一化；空→0；完全相等→1；互为子串→0.96；否则用 right 每字符在 left 里包含计数 / max(len) 得 0~1 分。
- `findMatchedStudentFromSpeech(text)` 行3397：遍历学生取 score 最大者；`score>=CONFIG.AI_MATCH_THRESHOLD(0.62)` 才返回该学生，否则 null（未匹配）。
- `isStartRollCommand(text)` 行3411：归一化后命中 `includes('开始点名')` 或 (`includes('开始')&&includes('点名')`) 或 `==='点名开始'` 或 `==='开始抽选'`。

### 7.3 onresult 逻辑（行3514-3549）
1. 取 transcript → devLog“识别到文本”。
2. 若 `isStartRollCommand(transcript)`：showWarn“🎤 识别到「开始点名」，正在抽取...”→ `rollOneStudentByCurrentMode()`（按当前 currentRollMode 抽一人，见3.1），并 return。
3. 否则 `findMatchedStudentFromSpeech(transcript)`：
   - 命中 → devLog“识别到: X”，showWarn“✅ 识别到 X”，`selectStudentByName(matched.name,'voice')`。
   - 未命中 → devLog“未匹配”，临时把 warning 显示“🔍 听到: X (未匹配)” 1.5s 后还原。

### 7.4 selectStudentByName(name, source='voice') —— 行3416-3444
- 精确 `name` 匹配；找不到 → devLog“未找到学生”。
- 命中：`selectedCount+1`、`totalSelections+1`、`history.push({time,names:[name],source})`、save、`showResult([student])`、updateStats/renderList/renderDev、`confetti()`、`highlightStudentInList`；若 `currentRollMode==='ai'` 再 `increaseAIWeightForStudent(name)`；showWarn“🎉 点名成功：X”。
- history 的 `source` 值："voice"(selectStudentByName)、"voice_cmd"(rollOne)、"ai"(runAiRoll)、无(主按钮 start)。

---

## 8. DataManager.save(appData) 的全部调用时机（逐处）
> 凡学生/抽取数据变更，几乎都会保存。除下列显式调用外，`setProbabilityMode()`(行3654) 与 `switchToRollMode()`(行2645) 内部自带 save；而许多上层操作会先调 setProbabilityMode 再 save，形成双重保存。

| 行 | 所在函数 | 时机 |
|---|---|---|
| 2589 | `increaseAIWeightForStudent` | AI 模式语音点名提权 +0.1 |
| 2645 | `switchToRollMode` | 切换 normal/ranking/ai（ai 先清 weight=1.0）|
| 2688 | `rollOneStudentByCurrentMode` | 语音“开始点名”抽一人落定 |
| 3213 | `updateWeightDisplay` | 每次拖拽/滑块/快捷按钮改某生权重 |
| 3317 | `renderIdCards` 内事件 | 改某生身份证号 |
| 3360 | `runAiRoll` | AI 点名多人落定 |
| 3431 | `selectStudentByName` | 语音点名到具体姓名 |
| 3604 | `start()` | 主按钮 pick 模式动画结束、weightedSelect 落定后 |
| 3619 | `saveNames` | 保存编辑后的名单（前有 setProbabilityMode）|
| 3620 | `resetStats` | 重置 selectedCount/totalSelections/history 后（前有 setProbabilityMode）|
| 3654 | `setProbabilityMode` | 切换/初始化权重分配（含 init 行4822 首屏调用）|
| 3697 | `confirmImp` | JSON 导入成功后 |

- `history.push` 仅行 2687 / 3359 / 3430 / 3603，对应 voice_cmd / ai / voice(或自定义 source) / 主按钮无 source 四种来源。
- `StorageManager` 其它 setItem：debug 开关(1912/1966/1955读)、WhatsNew(4705/4746/4756)、birthday 开关(4795)、OpenRouter key/model(4810/4817)、clear(3999/3025)。

---

## 附：复现必须留意的一致性细节（易错清单）
1. weight 双尺度（0.1~10 原始值 vs weight×10 显示%）切勿混为一谈；抽中概率是 weight/Σweight 另有计算。
2. 默认/normal 权重 = 10，`resetAllW` 却给 1.0，AI 模式给 1.0 —— 三个来源写不同基线，请照抄含 UI 外观差异。
3. init 首屏必调 `setProbabilityMode(false)` 覆盖 defaultStudents 里的 9.9；之后一切 save/reset/import 也先 setProbabilityMode。
4. `globalMaxStudents` 仅内存、默认 500、不落盘。
5. LOWEST_STUDENTS=[杨子霂,骆清昊] 恒为 0.1（两个名字不在默认 49 人中）。
6. AI 点名按钮实为“加权随机+自适应提权”，非联网选人；真正的 OpenRouter 调用只在 AI 聊天抽屉(sendChatMessage/submitAIDemand)。
7. agent 工具 open_student_edit/edit_students/clear_selected 引用未定义函数，属潜在失效，不影响 UI 主流程。
8. main 按钮(weightedSelect，无兜底) 与 rollOne/aiWeightedSelect(有 Math.max(0.01,·) 兜底) 的抽取略有差异。
9. MAX_FAILS=15 次密码失败触发 selfDestruct；元防护会 clear 存储并关窗，重实现勿删。
