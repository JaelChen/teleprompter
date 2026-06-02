# 交接文档 / 修复清单(给 Codex / GPT 直接读取)

> **读我顺序**:先读 `ARCHITECTURE.md`(权威规格),再读本文件(本轮要做的具体改动)。
> 本文件由架构师(Claude)在审完第一版代码后产出。**你(Codex/GPT)负责按本清单改 `index.html`**;
> 架构师负责设计与审查,不直接写业务逻辑代码。改完请逐条对照「验证」自测。
> 最后更新:2026-06-02。

---

## 0. 当前状态

- ✅ **第一轮修复(§2:P0 黑底白字 / P1 进入暂停 / P1 末行 50dvh)已由 GPT 完成,架构师验收通过。**
- 🆕 **第二轮新增需求见 §5,请实现。** 这是用户明确批准的范围扩展,已写入 `ARCHITECTURE.md` §1 功能范围。
- 唯一代码文件:`index.html`(单文件,内联 CSS + 原生 JS,无框架、无构建、双击即开)。
- 第一版功能已实现:贴稿 / 匀速滚动(开始·暂停·继续·重置)/ 镜像翻转 / 字号实时调 / 速度实时调 / Wake Lock 屏幕常亮 / 仅稿子文本持久化。
- **编辑主页已由架构师重设计并写入**(深色简约风,见 `ARCHITECTURE.md` §10)。**此部分视觉规范请勿改动。**
- 本轮你要做的,是下面 §2 的修复清单(其中 **P0 配色变更是用户明确要求,必须做**)。

---

## 1. 架构师代码评审反馈(结论)

整体实现质量好,忠实于架构文档。以下为审查结论:

**✅ 实现正确、勿动:**
- 镜像与位移分层:`scaleX(-1)` 在 `.viewport`、`translate3d` 在 `.scroller`,互不干扰。
- rAF + delta-time 匀速,`dt` 上限 0.08s 钳制,防切后台跳变。
- Wake Lock 申请/释放 + `visibilitychange` 回前台重申请。
- 仅持久化稿子文本、防抖写入、try/catch 降级。
- 速度指数映射(8→180 px/s)。

**⚠️ 需修复(见 §2 清单):**
- P0:提词页配色需改为黑底白字(用户要求)。
- P1:进入提词即自动开滚,体验欠佳。
- P1:滚到底时最后一行到不了阅读基准线。

---

## 2. 修复清单(按优先级)

### 🔴 P0 — 提词页改「黑底白字」(用户明确要求,必须做)

**背景**:原规格是白底黑字,**用户已于 2026-06-02 评审后改为黑底白字**(理由:反光玻璃提词器场景,黑底防眩光、对比最高,是行业标准做法)。`ARCHITECTURE.md` 已同步更新。
**注意**:这是**整页配色反转**,不仅是背景色——控制栏、控件、滑块、基准线都要连带改成深色主题,保证可读与可点。**只改提词模式(`.prompter` 及其内部 + `.controls`),不要动编辑主页 `.editor` 的任何样式。**

**具体改法(`<style>` 内的 `:root` 与控制栏区块,精确替换):**

1) `:root` 中提词相关变量:
```css
/* 改前 */
--prompter-bg: #ffffff;
--prompter-text: #000000;
--control-bg: rgba(255, 255, 255, 0.88);
--control-border: rgba(0, 0, 0, 0.14);
--control-text: #111111;

/* 改后 */
--prompter-bg: #000000;
--prompter-text: #ffffff;
--control-bg: rgba(18, 18, 20, 0.86);     /* 深色玻璃控制栏 */
--control-border: rgba(255, 255, 255, 0.16);
--control-text: #f2f2f2;
```

2) 控制栏内的按钮/控件底色(原本是浅灰系,需改深色系):
```css
/* .control-button  改前: background:#f2f2f2; :active background:#e3e3e3; */
.control-button       { background: rgba(255, 255, 255, 0.12); }
.control-button:active{ background: rgba(255, 255, 255, 0.22); }

/* .control-group  改前: background:#f7f7f7; border:1px solid rgba(0,0,0,0.08); */
.control-group        { background: rgba(255, 255, 255, 0.08); border: 1px solid rgba(255, 255, 255, 0.14); }

/* .mirror-toggle  改前: background:#f7f7f7; border:1px solid rgba(0,0,0,0.08); */
.mirror-toggle        { background: rgba(255, 255, 255, 0.08); border: 1px solid rgba(255, 255, 255, 0.14); }

/* 滑块/复选框强调色 改前: accent-color:#111111; */
.control-group input[type="range"] { accent-color: #ffffff; }
.mirror-toggle input               { accent-color: #ffffff; }
```

3) 阅读基准线在纯黑背景上提高可见度(可选但建议):
```css
/* .guide  改前: background: rgba(25,195,125,0.55); */
.guide { background: rgba(33, 192, 138, 0.75); }
```

**验证**:点「开始提词」进入后,整屏纯黑、文字纯白大字;底部控制栏为深色玻璃、按钮/滑块/镜像开关清晰可点;镜像、滚动、基准线一切正常;编辑主页配色不受影响。

---

### 🟠 P1 — 进入提词不要自动开滚,改为「进入即暂停」

**现状**:`mode.enterPrompter()` 末尾直接 `scrollEngine.play()`,一进去就开始滚。
**期望**:进入提词后**停在开头**,首行落在基准线附近,用户按「开始/继续」或点正文区再滚(更符合真实提词,便于就位)。
**改法**:`enterPrompter()` 里把 `scrollEngine.play();` 去掉,只保留 `calculateBounds()` + `setStartPosition()`;确保播放按钮初始显示「开始」。同时把 `dom.playButton` 初始/进入时文案设为「开始」,`scrollEngine.play()` 内已会改成「暂停」。
**验证**:进入提词为静止状态,首行在中部基准线附近;点一下正文或按钮才开始匀速滚动。

---

### 🟠 P1 — 滚到底时最后一行能到达阅读基准线

**现状**:`.scroller` 底部 padding 是 `42dvh`,`maxDistance = contentHeight - stageHeight`,滚到底时末行停在中线下方,够不到基准线(50%)。
**期望**:末行能滚到基准线(屏幕垂直中部)再停。
**改法**:把 `.scroller` 的 padding 底部值由 `42dvh` 调到约 `50dvh`(即 `padding: 52dvh 7vw 50dvh;`),让内容尾部留白等于"基准线到底部"的距离。改后注意复测 `calculateBounds()` 仍正确。
**验证**:一直滚到结束,最后一行恰好停在中部基准线高度,不会卡在下半屏。

---

### ⚪ 可选 — 编辑主页两个微交互的去留(用户定,默认保留)

编辑主页有 **字数统计**(右下角)和 **「草稿自动保存」胶囊**两个微交互,只读现有状态、未引入新功能。
若用户判定超出第一版范围,删除对应 DOM/CSS 与 `controls.updateCount` 即可,不影响任何核心逻辑。**未经用户指示不要擅自删。**

---

## 3. 绝对不要改动的部分

- **编辑主页视觉规范**(`ARCHITECTURE.md` §10):配色令牌、品牌栏、纸面输入、CTA 按钮、横屏适配——架构师设计,勿动。
- **镜像分层**(`.viewport` 镜像 / `.scroller` 位移分离)。
- **滚动引擎**(rAF + delta-time + `dt` 钳制)。
- **Wake Lock**(申请/释放 + `visibilitychange` 重申请)。
- **持久化范围**:只允许持久化稿子文本一项,不要持久化字号/速度/镜像。
- 不要新增任何 `ARCHITECTURE.md`「功能范围」之外的功能。

---

## 4. 改完总验证清单

- [ ] 提词页:纯黑底、纯白大字;控制栏深色玻璃、控件清晰可点。
- [ ] 编辑主页配色与布局完全未受影响。
- [ ] 进入提词为静止态,首行在基准线附近;手动开始后匀速滚动。
- [ ] 滚到结尾,末行能到达中部基准线。
- [ ] 镜像一键翻转正常;字号、速度滚动中实时可调。
- [ ] 提词时屏幕常亮;切后台回前台仍常亮。
- [ ] 稿子刷新后自动恢复;无其他持久化。
- [ ] 单文件、双击即开、无外部依赖、无构建步骤。

---

## 5. 第二轮迭代(新增需求,用户 2026-06-02 提出 —— 请实现)

本轮新增两个功能。**这是用户明确批准的范围扩展**,已写入 `ARCHITECTURE.md` §1(功能项 8、9)。
两项都只作用于**提词模式**,不要改动编辑主页。所有新设置**仍不持久化**(只持久化稿子文本这条铁律不变)。

---

### 🆕 A —「文字宽窄(左右边距)」可调

**背景**:横屏远距离念稿时,若文字铺满整屏太宽,眼睛要左右大幅扫视,出镜不美观。需要一个控件把**文字栏缩窄**(等价于加大文字与边框的左右边距)。

**实现要点(改 `.scroller` 的左右内边距,用 CSS 变量驱动,便于实时调):**

1) `.scroller` 的 padding 改为用变量控制左右:
```css
/* 改前: padding: 52dvh 7vw 50dvh; */
.scroller {
  --side-pad: 7vw;                 /* 默认值=原值,保证不调也合理 */
  padding: 52dvh var(--side-pad) 50dvh;
}
```

2) 控制栏新增一个滑块(放在「速度」「字号」之后、「镜像」之前):
```html
<label class="control-group">
  宽窄
  <input type="range" min="5" max="30" step="1" value="7" data-role="margin">
</label>
```
- 语义:**数值越大 → 左右边距越大 → 文字越窄**(滑块从左到右 = 宽→窄,直觉一致)。
- 范围 `5vw–30vw`,默认 `7vw`(=改造前的观感)。

3) JS:
- `state` 增加 `sideMargin: 7`。
- 新增 `controls.applyMargin()`:
  ```js
  applyMargin() {
    dom.scroller.style.setProperty('--side-pad', `${state.sideMargin}vw`);
    requestAnimationFrame(() => scrollEngine.calculateBounds()); // 宽度变→换行变→内容高度变,必须重算边界
  }
  ```
- `dom` 增加 `marginInput`;`bind()` 里监听 `input`:`state.sideMargin = Number(value); this.applyMargin();`
- `init()` 里:`state.sideMargin = Number(dom.marginInput.value); controls.applyMargin();`

**验证**:提词页拖动「宽窄」滑块,文字栏实时变窄/变宽且仍居中;变窄后滚动到底末行仍能到达基准线(因已 `calculateBounds`);镜像开启时左右边距对称、不偏移。

---

### 🆕 B —「进入先调设置 → 点开始 → 5 秒倒计时 → 再播放」

**目标流程**:
1. 点编辑页「开始提词」→ 进入黑底白字提词页,**静止停在开头,不自动播放**(第一轮已实现"进入即暂停",本轮在其上加倒计时)。
2. 在提词页**先调设置**:镜像开关 / 字号 / 宽窄 / 速度(控制栏都在,静止时即可调)。
3. 调好后点「开始」→ **全屏 5 秒倒计时**(显示 5→4→3→2→1)→ 倒计时结束自动开始匀速滚动。
4. **倒计时只在"首次开始"出现**(用户已确认):播放中暂停后点「继续」**立即接着读、不再倒计时**;点「重置」回到待命态,下次「开始」重新倒计时。

**状态机**:`state` 增加 `phase: 'ready' | 'counting' | 'playing' | 'paused'`,作为提词播放的唯一真值来源。

| phase | 含义 | 播放按钮文案 |
|---|---|---|
| `ready` | 进入提词 / 重置后,静止在开头 | `开始` |
| `counting` | 5 秒倒计时中 | `取消` |
| `playing` | 正在匀速滚动 | `暂停` |
| `paused` | 播放中被暂停 | `继续` |

**主操作(播放按钮点击 与 点正文区 `dom.stage` 共用同一处理)按 phase 分发:**
- `ready` → 开始倒计时(进入 `counting`)
- `counting` → 取消倒计时,回到 `ready`(位置不变,仍在开头)
- `playing` → 暂停(进入 `paused`)
- `paused` → 立即继续(进入 `playing`,**不倒计时**)

**倒计时实现建议**(新增一个 `countdown` 模块,5 秒 = 常量,便于日后改):
```js
const countdown = {
  timerId: 0, remaining: 0,
  start() {
    state.phase = 'counting';
    this.remaining = 5;
    dom.playButton.textContent = '取消';
    dom.countdown.textContent = this.remaining;
    dom.countdown.classList.remove('hidden');
    wakeLock.request();                       // 倒计时期间也保持常亮
    this.timerId = setInterval(() => {
      this.remaining -= 1;
      if (this.remaining <= 0) {
        this.stop();                          // 清计时器 + 隐藏数字
        state.phase = 'playing';
        scrollEngine.play();                  // 开始滚动(内部会把按钮置为"暂停")
      } else {
        dom.countdown.textContent = this.remaining;
      }
    }, 1000);
  },
  cancel() { this.stop(); state.phase = 'ready'; dom.playButton.textContent = '开始'; wakeLock.release(); },
  stop()   { clearInterval(this.timerId); this.timerId = 0; dom.countdown.classList.add('hidden'); },
};
```

**倒计时浮层 DOM**(放进 `.prompter__stage`,作为 `.viewport`/`.guide` 的兄弟,盖在最上层):
```html
<div class="countdown hidden" data-role="countdown" aria-hidden="true">5</div>
```
**CSS**(复用已有的全局 `.hidden`{display:none!important} 来显隐):
```css
.countdown {
  position: absolute; inset: 0; z-index: 3;
  display: grid; place-items: center;
  pointer-events: none;                 /* 让点击穿透到 stage,以便"取消" */
  color: var(--prompter-text);
  font-size: 28vh; font-weight: 800;
  font-variant-numeric: tabular-nums;   /* 数字等宽,跳秒不抖动 */
  background: rgba(0, 0, 0, 0.35);
}
```

**与现有函数的衔接 / 改动点**:
- `dom` 增加 `countdown: document.querySelector('[data-role="countdown"]')`。
- `mode.enterPrompter()`:进入后设 `state.phase = 'ready'`、按钮文案 `开始`(已有,补 phase 赋值)。
- 暂停/继续:`playing→paused` 走 `scrollEngine.pause()`(按钮"继续")+ `wakeLock.release()`;`paused→playing` 走 `scrollEngine.play()`(按钮"暂停")。把 `state.phase` 同步设好。
- `scrollEngine.reset()`:先 `countdown.stop()`,再停播放、回开头,设 `state.phase='ready'`、按钮 `开始`、`wakeLock.release()`。(这同时修掉了第一轮遗留的"重置后按钮显示继续"小问题。)
- `mode.exitPrompter()`:若在 `counting` 先 `countdown.cancel()`;若在 `playing` 先暂停;再退出。确保不会带着定时器离开。
- `scrollEngine.tick()` 滚到底:停下并设 `state.phase='paused'`(按钮"继续")、`wakeLock.release()`。

**控制栏布局注意(新增"宽窄"后控件变 7 个,横屏要放得下):**
给 `.controls` 加 `flex-wrap: wrap;` 和一点 `row-gap`,并在 `@media (orientation: landscape)` 里适当缩小各控件 `min-width`,允许在窄屏下**换成两行**,避免横向溢出/被裁。务必保证每个控件仍可点。

**验证**:
- [ ] 进入提词:静止在开头,按钮显示「开始」,可先调镜像/字号/宽窄/速度。
- [ ] 点「开始」:出现全屏 5→1 倒计时,期间屏幕常亮、不滚动;倒计时中按钮显「取消」,点一下(或点屏幕)可取消回到待命。
- [ ] 倒计时结束:自动开始匀速滚动,按钮变「暂停」。
- [ ] 播放中暂停→点「继续」:立即接着滚,**不再倒计时**。
- [ ] 「重置」:回到开头与待命态,按钮「开始」,再次「开始」会重新倒计时。
- [ ] 退出提词:无残留定时器(不会回到编辑页还在偷偷计时)。

---

### ⚪ 顺带的小优化(可一并做)
- `.guide` 的 `box-shadow` 颜色统一为 `rgba(33, 192, 138, …)`(当前还残留旧的 `25, 195, 125`,纯色值不一致)。
- (可选)`.scroller` 顶部 padding `52dvh` 改 `50dvh`,与底部对称,使首行/末行都精确落在基准线。

---

## 6. 第三轮迭代(提词界面重构,用户 2026-06-02 提出 —— 请实现)

> **背景**:真机实测发现底部控制栏在手机上**占屏比例太大**,挤掉了正文。
> 参考主流提词器,改为「**顶部极简图标条 + 按需弹出的设置面板**」范式:提词正文几乎占满全屏,设置藏进面板。
> 本轮只动**提词模式**;编辑主页、镜像分层、滚动引擎、Wake Lock、持久化范围一律不动。新增设置仍**不持久化**。

### 6.0 新流程总览

```
编辑页 →[点"开始提词"]→ 弹出「横屏 / 竖屏」选择
        →[选其一]→ 进入提词页(黑底白字, 文字静止在开头)
                   且【自动弹出设置面板】(phase=ready, 不自动播放)
        →[在面板里调: 镜像/提示线/倒计时档位/速度/字号/宽窄, 可点"重置"]
        →[点面板外的区域]→ 关闭面板, 进入干净提词视图(只剩右上角 4 个图标)
        →[点右上角"开始"]→ 按所选秒数倒计时(选"关闭"则立即)→ 匀速滚动
```

### 6.1 新增 / 变化的 state

```js
// 在现有 state 上新增:
orientation:  'landscape',  // 'landscape' | 'portrait',进入提词时用户选择
countdownSec: 5,            // 0 | 3 | 5 | 10(0 = 关闭, 立即开始)
showGuide:    true,         // 提示线(中部基准线)开关
settingsOpen: false,        // 设置面板是否打开
```
保留现有:`phase / isPlaying / isMirrored / fontSize / speed / sideMargin / position`。

### 6.2 删除 / 迁移的旧结构

- **删除底部 `<nav class="controls">` 整条**(及其 CSS),它就是"占比太大"的根源。
- 原来栏里的控件(速度/字号/宽窄/镜像)**迁移进设置面板**(§6.5)。
- 原 `dom.playButton` 文案驱动逻辑改由右上角图标 + 面板承接(§6.3)。

### 6.3 右上角图标条(4 个,提词页常驻)

容器固定在右上角,**用安全区内边距避开刘海/状态栏**:`padding-top: env(safe-area-inset-top); padding-right: env(safe-area-inset-right);` `z-index` 高于正文、低于面板背板。每个图标 ≥40px 点击区、半透明深色圆底 + 白色图形(可用内联 SVG 或字符)。

| 图标 | 含义 | 行为 |
|---|---|---|
| ←(返回) | 退出 | `mode.exitPrompter()` 回编辑页(清定时器、释放 WakeLock) |
| ⚙(设置) | 打开/关闭面板 | `openSettings()`:`settingsOpen=true`;**若 phase==='playing' 先暂停**;再点或点面板外则 `closeSettings()` |
| ▶(开始) | 开始/继续 | `ready` → 倒计时(`countdownSec`)后播放;`paused` → **立即继续(不倒计时)**;`counting/playing` → 忽略 |
| ⏸(暂停) | 暂停 | `playing` → 暂停;其他忽略 |

> 「开始」「暂停」按用户要求是**两个独立图标常驻**。可保留"点正文区切换播放/暂停"作为便捷手势(仅在面板关闭时生效;面板打开时点正文外区域=关闭面板)。

### 6.4 横屏 / 竖屏选择 + 旋转提示

- 点「开始提词」后,先弹一个轻量选择层(两个大按钮:**横屏 / 竖屏**,可带"取消")。选定后 `state.orientation` 定下,调 `mode.enterPrompter()`。
- 提词正文本就用 `dvh/vw` 自适应,**两种朝向同一套滚动逻辑**,无需为竖屏另写引擎。
- **旋转提示**:把原"请横屏"遮罩改为按"所选朝向 vs 设备实际朝向"显示,文案动态:
  ```css
  /* 选了横屏但设备竖着 → 提示转横屏 */
  @media (orientation: portrait) {
    body[data-mode="prompter"][data-orient="landscape"] .rotate-tip { display: grid; }
  }
  /* 选了竖屏但设备横着 → 提示转竖屏 */
  @media (orientation: landscape) {
    body[data-mode="prompter"][data-orient="portrait"] .rotate-tip { display: grid; }
  }
  ```
  JS 在进入时 `document.body.dataset.orient = state.orientation`,并按朝向设置提示文案("请横屏使用" / "请竖屏使用")。**编辑主页永不显示该提示**(沿用现有 `data-mode` 限定)。

### 6.5 设置面板(二级设置框)内容与交互

**触发**:进入提词自动打开;之后由右上角 ⚙ 开关。
**关闭**:点面板**背板(面板以外区域)**关闭;面板内部点击不关闭(背板与面板分层,点击事件只在背板上触发关闭)。
**打开即暂停**:`openSettings()` 若正在播放则先 `controls.pause()`(用户已确认)。关闭后停在当前位置,等用户点「开始」。

**面板内容(只放我们现有的功能,不要引入参考图里的 循环/视野聚焦/录制/颜色 等本版不做的项):**

| 控件 | 类型 | 绑定 | 备注 |
|---|---|---|---|
| 文字镜像 | 开关 | `state.isMirrored` → `.viewport.is-mirrored` | 同现有逻辑 |
| 提示线 | 开关 | `state.showGuide` → 显隐 `.guide` | 默认开;关则隐藏中部基准线 |
| 倒计时 | 分段选择 关闭/3秒/5秒/10秒 | `state.countdownSec`(0/3/5/10) | 见 §6.6 |
| 播放速度 | 滑块 | `state.speed`(指数映射不变) | 实时 |
| 字号大小 | 滑块 | `state.fontSize` + `calculateBounds()` | 实时 |
| 宽窄 | 滑块 | `state.sideMargin` + `calculateBounds()` | 实时 |
| 重置 | 按钮 | `scrollEngine.reset()` | 回开头 + phase=ready |

> 实时调节的 `input` 监听沿用第二轮已有的写法,只是 DOM 容器从底栏换成面板。

### 6.6 倒计时档位(关闭 / 3 / 5 / 10 秒)

- 面板分段控件,选中写入 `state.countdownSec`(`关闭=0`)。
- `countdown.start()` 改为读 `state.countdownSec`:**若为 0 直接 `scrollEngine.play()`、不显示浮层**;否则 `remaining = state.countdownSec` 走原倒计时。
- "仅首次开始倒计时、继续不倒计时"的规则不变。

### 6.7 提示线开关

- `state.showGuide` 控制 `.guide` 显隐(`classList.toggle('hidden', !showGuide)`)。默认 `true`。

### 6.8 视觉设计指引(深色玻璃面板 + 翡翠绿,**勿照搬参考图的白底蓝**)

参考图是白底蓝色;但本应用提词页是**纯黑**、且常用于**反光玻璃/拍摄**,大白panel会眩光、出戏。故面板采用**深色玻璃**风格,与编辑页/品牌色统一:

- **背板**:`rgba(0,0,0,0.45)`,铺满,点击关闭。
- **面板**:`max-width: min(560px, 92vw)`;`background: rgba(22,24,28,0.92)` + `backdrop-filter: blur(12px)`;圆角 `20px`;`border: 1px solid rgba(255,255,255,0.08)`;大投影。横屏居中、竖屏可贴下半屏,内部用响应式网格,两种朝向都不溢出。
- **开关磁贴(镜像/提示线)**:圆角磁贴,图标+文字+开关;开启态用强调色 `#21c08a`。
- **分段倒计时**:胶囊分段,选中段填 `#21c08a`。
- **滑块**:左标签 + 右数值,`accent-color: #21c08a`。
- **重置**:描边按钮(浅色边、透明底)。
- **右上角图标**:`rgba(255,255,255,0.14)` 圆底、白色图形、`:active` 加深;图标可用内联 SVG(←、齿轮、▶ 三角、⏸ 双竖条)。

> 设计令牌沿用 `ARCHITECTURE.md` §10:强调绿 `#21c08a`/`#16a877`,深色玻璃,系统字体。**编辑主页视觉不变。**

### 6.9 验证清单

- [ ] 提词正文几乎占满全屏,**不再有占比过大的底部控制栏**。
- [ ] 点「开始提词」先出现 横屏/竖屏 选择;选竖屏能竖屏提词、不再被"请横屏"打扰;选横屏且设备竖着才提示转横屏(反之亦然)。
- [ ] 进入提词**自动弹出设置面板**,文字静止;点**面板外**关闭面板进入干净视图。
- [ ] 右上角 4 图标:返回回编辑;设置可重开面板且**自动暂停**;开始按所选倒计时(关闭=立即)后播放;暂停可暂停。
- [ ] 面板内 镜像/提示线/倒计时/速度/字号/宽窄/重置 全部生效,滑块实时。
- [ ] 提示线开关能隐藏/显示中部基准线。
- [ ] 倒计时档位 关闭/3/5/10 均正确;继续(paused→播放)不倒计时。
- [ ] 退出/重置无残留定时器;镜像、滚动、持久化等旧功能零回归。
- [ ] 面板为深色玻璃风,黑底提词页无大面积白光;编辑主页视觉未变。
