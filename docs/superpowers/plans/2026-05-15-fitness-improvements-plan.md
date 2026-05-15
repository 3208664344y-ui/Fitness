# 健身日历改进实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复组数切换抖动bug + 实现动作分类标签页系统

**Architecture:** 所有改动集中在 `fitness.html` 单文件内。CSS修复通过 min-height 稳定弹窗高度。动作分类用横向标签栏 + 2列网格替代原有 `<select>` 下拉，新增 `CATEGORIES` 数据结构，`PRESETS` 兼容数组从中动态生成。

**Tech Stack:** 纯 HTML/CSS/JS，localStorage 数据层不变

---

### Task 1: 修复组数切换布局抖动

**Files:**
- Modify: `fitness.html:166-170`（.sheet 样式）
- Modify: `fitness.html:197-199`（.set-detail-list 样式）

- [ ] **Step 1: 给 .sheet 添加 min-height**

在 `.sheet` 样式块中添加 `min-height: 420px;`：

```css
.sheet {
  background: #fff;
  border-radius: 20px 20px 0 0;
  padding: 24px 20px; width: 100%; max-width: 500px;
  max-height: 90vh; overflow-y: auto;
  animation: slideUp 0.2s ease;
  min-height: 420px;   /* 新增：固定最小高度，防止组数变化时弹窗抖动 */
}
```

- [ ] **Step 2: 给 .set-detail-list 添加 min-height**

```css
.set-detail-list {
  margin-bottom: 12px;
  min-height: 180px;   /* 新增：固定最小高度，组数减少时不会塌陷 */
}
```

- [ ] **Step 3: Commit**

```bash
git add fitness.html
git commit -m "fix: stabilize sheet height when changing set count"
```

---

### Task 2: 替换 PRESETS 为 CATEGORIES 数据结构

**Files:**
- Modify: `fitness.html:392-403`（PRESETS 定义和选项填充逻辑）

- [ ] **Step 1: 删除 PRESETS 数组和 presetSelect 填充代码**

删除以下两段代码：

```js
// 删除这段 (行 392-398)
const PRESETS = [
  '卧推', '深蹲', '硬拉', '引体向上', '推举', '划船', '弯举',
  '臂屈伸', '卷腹', '平板支撑', '飞鸟', '侧平举', '腿举',
  '臀推', '高位下拉', '双杠臂屈伸', '俯身飞鸟', '面拉'
];

// 删除这段 (行 399-403)
const presetSelect = document.getElementById('exSelect');
PRESETS.forEach(p => {
  const opt = document.createElement('option');
  opt.value = p; opt.textContent = p;
  presetSelect.appendChild(opt);
});
```

- [ ] **Step 2: 添加 CATEGORIES 数据结构**

在原 PRESETS 位置添加：

```js
const CATEGORIES = {
  '胸':       ['卧推','上斜卧推','下斜卧推','哑铃飞鸟','绳索夹胸','双杠臂屈伸','俯卧撑','哑铃卧推'],
  '背':       ['引体向上','划船','高位下拉','俯身飞鸟','面拉','单臂哑铃划船','硬拉','T杆划船'],
  '腿':       ['深蹲','腿举','臀推','前蹲','保加利亚分腿蹲','腿弯举','腿屈伸','罗马尼亚硬拉'],
  '二头':     ['弯举','锤式弯举','牧师凳弯举','集中弯举','上斜哑铃弯举'],
  '三头':     ['臂屈伸','窄距卧推','绳索下压','法式弯举','俯身臂屈伸'],
  '肩':       ['推举','侧平举','前平举','阿诺德推举','俯身侧平举'],
  '核心':     ['卷腹','平板支撑','悬垂举腿','俄罗斯转体','死虫式'],
  '热身':     ['开合跳','肩关节环绕','髋关节激活','跳绳'],
  '拉伸':     ['股四头肌拉伸','腘绳肌拉伸','胸大肌拉伸','背阔肌拉伸','肩部拉伸','髋屈肌拉伸'],
  '拳击':     ['空击训练','打靶','重沙袋','速度球'],
  'CrossFit': ['药球翻站','波比跳','墙球','双摇跳绳','抓举'],
  'HyperX':   ['雪橇推','雪橇拉','农夫行走','战绳','翻轮胎'],
};

// 兼容：从 CATEGORIES 扁平生成所有动作列表
const PRESETS = [].concat(...Object.values(CATEGORIES));

// 分类列表（保持顺序，用于标签栏渲染）
const CATEGORY_KEYS = Object.keys(CATEGORIES);
```

- [ ] **Step 3: 清理对 presetSelect 的引用**

后续代码中 `sel.value` 和 `customInput` 的校验逻辑会改用新的分类系统，暂时保留 `sel` 和 `customInput` 变量声明（它们在 Task 5 中会重新接线）。

搜索 `presetSelect` 确认无残留引用：
```bash
grep -n "presetSelect" fitness.html
```
预期：无结果。

- [ ] **Step 4: Commit**

```bash
git add fitness.html
git commit -m "refactor: replace PRESETS array with CATEGORIES data structure"
```

---

### Task 3: 更新弹窗 HTML 结构

**Files:**
- Modify: `fitness.html:316-341`（弹窗 sheet 内容）

- [ ] **Step 1: 替换 select 和自定义输入区域**

将现有的：

```html
<div class="form-group">
  <label>动作名称</label>
  <select id="exSelect"><option value="">选择预设动作...</option></select>
</div>
<div class="form-group">
  <label>或输入自定义动作</label>
  <input type="text" id="exCustom" placeholder="输入动作名称">
</div>
```

替换为：

```html
<div class="form-group">
  <label>选择动作</label>
  <div class="cat-tabs" id="catTabs"></div>
  <div class="ex-grid" id="exGrid"></div>
  <input type="text" id="exCustom" placeholder="或输入自定义动作名称" style="display:none; margin-top: 8px;">
</div>
```

注意：`sel` 元素被完全移除。删除 `const sel = document.getElementById('exSelect');` 相关引用。`customInput` 保留但初始隐藏。

- [ ] **Step 2: Commit**

```bash
git add fitness.html
git commit -m "feat: add category tabs and exercise grid HTML structure"
```

---

### Task 4: 添加新 CSS 样式

**Files:**
- Modify: `fitness.html`（`<style>` 块，在 `.form-actions` 之前）

- [ ] **Step 1: 添加分类标签栏样式**

在 `.volume-preview` 样式块之后、`.form-actions` 之前插入：

```css
/* 分类标签栏 */
.cat-tabs {
  display: flex; gap: 4px; overflow-x: auto;
  padding-bottom: 10px;
  -webkit-overflow-scrolling: touch;
  scrollbar-width: none;
}
.cat-tabs::-webkit-scrollbar { display: none; }
.cat-tab {
  padding: 7px 14px; border-radius: 16px;
  font-size: 13px; font-weight: 600; white-space: nowrap;
  background: #f3f4f6; color: #6b7280;
  cursor: pointer; border: none;
  transition: all 0.15s;
}
.cat-tab.active { background: #4f46e5; color: #fff; }

/* 动作网格 */
.ex-grid {
  display: grid; grid-template-columns: 1fr 1fr;
  gap: 8px; max-height: 200px; overflow-y: auto;
  margin-bottom: 8px;
}
.ex-tile {
  padding: 12px 10px; border-radius: 12px;
  background: #f9fafb; font-size: 14px; font-weight: 600;
  text-align: center; cursor: pointer;
  border: 2px solid transparent;
  transition: all 0.15s;
}
.ex-tile:hover { background: #eef2ff; }
.ex-tile.selected {
  background: #4f46e5; color: #fff;
  border-color: #4f46e5;
}
```

- [ ] **Step 2: Commit**

```bash
git add fitness.html
git commit -m "style: add category tabs and exercise grid CSS"
```

---

### Task 5: 添加分类切换和动作选择 JS 逻辑

**Files:**
- Modify: `fitness.html`（`<script>` 块）

- [ ] **Step 1: 添加状态变量和渲染函数**

在 `let editingId = null;` 之后添加：

```js
let currentCategory = CATEGORY_KEYS[0];
let selectedExercise = '';
```

在 `openAddForm` 函数之前添加渲染函数：

```js
function renderCatTabs() {
  const container = document.getElementById('catTabs');
  container.innerHTML = CATEGORY_KEYS.map(cat => {
    const activeClass = cat === currentCategory ? ' active' : '';
    return `<span class="cat-tab${activeClass}" data-cat="${cat}">${cat}</span>`;
  }).join('') + '<span class="cat-tab" data-cat="__custom__">自定义</span>';

  container.querySelectorAll('.cat-tab').forEach(tab => {
    tab.addEventListener('click', () => {
      currentCategory = tab.dataset.cat;
      selectedExercise = '';
      document.getElementById('exCustom').value = '';
      renderCatTabs();
      renderExGrid();
    });
  });
}

function renderExGrid() {
  const grid = document.getElementById('exGrid');
  const customInput = document.getElementById('exCustom');

  if (currentCategory === '__custom__') {
    // 自定义模式：隐藏网格，显示文本输入
    grid.style.display = 'none';
    customInput.style.display = 'block';
    customInput.placeholder = '输入自定义动作名称';
    customInput.focus();
    return;
  }

  // 正常分类模式
  grid.style.display = 'grid';
  customInput.style.display = 'none';

  const exercises = CATEGORIES[currentCategory] || [];
  grid.innerHTML = exercises.map(name => {
    const selClass = name === selectedExercise ? ' selected' : '';
    return `<div class="ex-tile${selClass}" data-name="${escapeHtml(name)}">${escapeHtml(name)}</div>`;
  }).join('');

  grid.querySelectorAll('.ex-tile').forEach(tile => {
    tile.addEventListener('click', () => {
      selectedExercise = tile.dataset.name;
      document.getElementById('exCustom').value = selectedExercise;
      renderExGrid();
    });
  });
}
```

- [ ] **Step 2: 更新 openAddForm 函数**

修改 `openAddForm`，在打开时渲染分类标签和网格：

```js
function openAddForm() {
  editingId = null;
  sheetTitle.textContent = '添加训练';
  submitBtn.textContent = '确认添加';
  currentCategory = CATEGORY_KEYS[0];
  selectedExercise = '';
  document.getElementById('exCustom').value = '';
  setSetCount(3);
  renderCatTabs();
  renderExGrid();
  overlay.classList.add('show');
}
```

- [ ] **Step 3: 更新提交按钮的校验逻辑**

在提交按钮的 click handler 中，`name` 的获取改为直接用 `customInput.value`：

```js
document.getElementById('submitAdd').addEventListener('click', () => {
  let name = document.getElementById('exCustom').value.trim() || selectedExercise;
  if (!name) { alert('请选择或输入动作名称'); return; }
  // ... 其余逻辑不变
```

- [ ] **Step 4: 清理对 sel 的引用**

删除这行：
```js
const sel = document.getElementById('exSelect');
```

删除 `customInput` 和 `sel` 的事件监听器中与 `sel` 相关的部分。保留 `customInput` 的 input 事件但简化：

```js
// 删除原有的 (行 701-702):
customInput.addEventListener('input', () => { if (customInput.value) sel.value = ''; });
sel.addEventListener('change', () => { if (sel.value) customInput.value = ''; });

// 替换为：
customInput.addEventListener('input', () => {
  if (customInput.value) {
    selectedExercise = customInput.value;
    renderExGrid();
  }
});
```

- [ ] **Step 5: Commit**

```bash
git add fitness.html
git commit -m "feat: add category switching and exercise selection logic"
```

---

### Task 6: 编辑模式自动定位分类

**Files:**
- Modify: `fitness.html`（openEditForm 函数）

- [ ] **Step 1: 更新 openEditForm**

修改 `openEditForm`，根据 `entry.name` 查找所属分组：

```js
function openEditForm(entry) {
  editingId = entry.id;
  sheetTitle.textContent = '编辑训练';
  submitBtn.textContent = '确认修改';

  // 查找动作所属分类
  let foundCat = null;
  for (const [cat, exercises] of Object.entries(CATEGORIES)) {
    if (exercises.includes(entry.name)) { foundCat = cat; break; }
  }
  if (foundCat) {
    currentCategory = foundCat;
    selectedExercise = entry.name;
    document.getElementById('exCustom').value = '';
  } else {
    currentCategory = '__custom__';
    selectedExercise = '';
    document.getElementById('exCustom').value = entry.name;
  }

  setDetailList.innerHTML = '';
  entry.sets.forEach((set, i) => {
    addSetRow(i, set.reps, set.weight);
  });
  currentSetCount = entry.sets.length;
  setCountDisplay.textContent = currentSetCount;
  updateTotalVol();
  renderCatTabs();
  renderExGrid();
  overlay.classList.add('show');
}
```

- [ ] **Step 2: Commit**

```bash
git add fitness.html
git commit -m "feat: auto-locate category when editing exercise"
```

---

### Task 7: 功能验证

**Files:**
- Modify: `fitness.html`（确认无误）

- [ ] **Step 1: 验证修复 — 组数切换无抖动**

打开应用 → 点击"添加训练" → 多次点击组数 +/- → 确认弹窗不会上下移动。

- [ ] **Step 2: 验证新增训练流程**

分类标签可左右滑动 → 点击不同分类标签切换动作网格 → 点击动作按钮高亮 → 输入次数/重量 → 确认添加 → 日历上出现圆点 → 列表中显示记录。

- [ ] **Step 3: 验证自定义动作**

点击"自定义"标签 → 网格隐藏、输入框显示 → 输入不在列表中的动作名 → 确认添加成功。

- [ ] **Step 4: 验证编辑模式**

在列表中某条记录点"编辑"按钮 → 弹窗自动定位到该动作所属分类并高亮 → 修改 → 确认修改生效。

- [ ] **Step 5: 验证键盘导航和原有功能**

键盘 ← → 切换日期正常 → 删除记录正常 → "今天"按钮高亮逻辑正常。

- [ ] **Step 6: Commit (如有修复)**

```bash
git add fitness.html
git commit -m "chore: final verification tweaks"
```
